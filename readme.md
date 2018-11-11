# Single source of truth

The main benefit of having a single stream of output is to ensure there is only a single source of truth for the observer (the view). This means that the observer does not have to rely on external states or variables for information on how to render itself.

## UiModel

The way that Soompi designed the output is through a sealed class of `UiModel`.

Taking `FeedUiModel` for example:

``` Kotlin
sealed class FeedUiModel : BaseUiModel {

    object Loading : FeedUiModel() //When we are loading the feed data
    object LoadingNewPage : FeedUiModel()

    data class Success(val feedEntryList: List<FeedEntry>) : FeedUiModel()

    data class Failure(val throwable: Throwable) : FeedUiModel()
    data class FailureNewPage(val throwable: Throwable) : FeedUiModel()

    data class ReactionFailure(val throwable: Throwable) : FeedUiModel()
    object LoginNeeded : FeedUiModel()
}
```

The pattern used is to combine multiple emission of `UiModel`s in order to form a complete picture. For example a `ReactionFailure` is immediately followed by a `Success`, such that the observer is able to render a failure of a reaction, and then receive an `UiModel` with all the information needed to render the view fully. In case the view has to be recreated, it will still be able to render any previous existing data.

## Drawbacks

While the implementation works, it does have a few drawbacks.

1. Need to remember to emit multiple `UiModel`s, and in the correct sequence
2. It can be confusing when emitting the `UiModel`s, where a reader might be wondering why a `ReactionFailure` is immediately followed by a `Success`. Is it actually a failure or a success?
3. Since the `UiModel` is emitted immediately one after the other, there can be scenarios where some of these emission are not consumed by the observer, since the observer only gets the latest emission.

The design of the `UiModel` can potentially lead to some bugs when handling actions.

``` Kotlin
FeedActions.LoadNextPage -> 
        feedUseCase.loadNextPage(feedType)
                .flatMapObservable { (data, state) ->
                    when (state) {
                        is State.Success -> data?.let {
                            Observable.just(FeedUiModel.Success(it))
                        }
                        is State.Failure ->
                            Observable.just(
                                FeedUiModel.FailureNewPage(state.error),
                                FeedUiModel.Success(feedList.toList())
                            )
                    }
                }
                .startWith(FeedUiModel.LoadingNewPage)
```

During the time after `LoadingNewPage` is emitted, but while waiting for a `Success` or `FailureNewPage` + `Success` to be emitted, if the view decides to recreate itself, the `UiModel` that it will receive is actually `LoadingNewPage` which does not contain a `List<FeedEntry>`, and will be unable to render the list of entries.

In this `FeedUiModel` example, the observer will not have enough information to render itself completely for almost all the types of `UiModel`.

## State & Event

Due to having only a single stream of output, the implementation is forcing what are essentially side effects to be a part of the `UiModel`, and each `UiModel` contains only an incomplete part of the entire state.

To address these shortcomings, state should be separated from side effects, and thus there should be two streams of output, one for state and one for side effects.

- `State` - contains all persistent information reflecting a snapshot of the view state
- `Event` - fire and forget ephemeral side effects which does not affect the state

With side effects separated from the actual state, new state can be emitted whenever the state is updated, and side effects can also be freely emitted any time. There will be no need to remember to have multiple emission in a specific order.

Any emission from the `State` stream can also be guaranteed to be valid and can be used to render the entire view at any point in time.

### 2 streams? Double source of truth?

Although there are now two streams of output, the observer only relies on the `State` stream as the single source of truth to render the ui. The observer does not need anything else to reflect the current state of the view.

### Implementation

The `State` stream should allow the observer to be able to have the latest version of the state at any point in time. In practice it can be an `Observable<State>` backed by a `BehaviorSubject`, or it can be a `LiveData`.

The `Event` stream should emit side effect events when there is an active observer, and queue the events when there are no observers. The stream should also **_not_** replay any events that have been consumed an observer. In practice it can be an `Observable<Event>` backed by an [`UnicastWorkSubject`](https://github.com/akarnokd/RxJava2Extensions#unicastworksubject). There are currently no `LiveData` implementations for this.

## Example Scenario
There is a list of items and a loading indicator. A snackbar should be shown when the list fails to load.

``` Kotlin
data class State(
    val list: List<Item>,
    val isLoading: Boolean
)

sealed class Event {
    data class LoadDataFailure(val throwable: Throwable): Event()
}

// viewmodel

val stateStream: Observable<State>
val eventStream: Observable<Event>

// view 

vm.stateStream.subscribe { state ->
    renderEntireView(state) // submit list to adapter, show/hide loading indicator
}

vm.eventStream.subscribe { event ->
    handle(event) // show snackbar
}
```

---

# Functional programming

Kotlin supports functions as a first class citizens of the language, which makes it easy to write functional code. This is extremely helpful for reducing the need of having to maintain a mutable state and avoiding side effects.

## Mutable state

Going back to `Feed` example, the `FeedViewModel` maintains a value of a list of entries.

```
class FeedViewModel {
    ...

    private val feedList : MutableList<FeedEntry> = mutableListOf() // mutable state

    ...

    fun handleAction() {
        useCase.loadData()
            .flatMap {
                // gets data from external state outside the stream
                // map to `UiModel` using the data
            }
            .doOnNext {
                // jumps outside the stream to update external state
            }
    }
}
```

Whenever an action is received, work will be performed which includes adding new data or updating the existing data, before using the latest data to create a new `UiModel` to emit for the observer.

These work that will be performed are usually asynchrounous, and RxJava is used to model the data flow. However, the data flow is not always contained in the stream. The flow jumps out of the stream to modify the state before jumping back to the stream and creating a new `UiModel` based on the updated state outside of the stream.

## Reducer

We can make use of the concept of `reducer` to avoid the need of having to maintain an external state, by having a *pure function* without side effects. 

It works by writing code that describes the changes to make rather than directly mutating a state. The reducer returns a value by applying the supplied function to the given value.

``` Kotlin
class Reducer<T>(private val reducer: (T) -> T) {
    operator fun invoke(oldState: T): T = reducer(oldState)
}

// Simple example

val plusOneReducer = Reducer<Int> { count -> count + 1 }
val timesTwoReducer = Reducer<Int> { count -> count * 2 }

val a = plusOneReducer(1) // a = 2
val b = timesTwoReducer(2) // b = 4

// State example

data class State(val items: List<Item>, val status: Status)

val loadingReducer = Reducer<State> {
    it.copy(status = Status.Loading)
}

val loadSuccessReducer = Reducer<State> { 
    it.copy(items = listOf(...), status = Status.Success) 
}

val loadFailureReducer = Reducer<State> {
    it.copy(isError = Status.Error)
}

val initialState = State(items = emptyList(), status = Status.Success)

val loadingState = loadingReducer(initialState)
val successState = loadSuccessReducer(loadingState)
val errorState = loadFailureReducer(loadingState)
```

Being functional with no side effects results in code being consistent and predictable. The output is guaranteed to be the same for a given input.

## Reducer within a stream

It is easy to generate a new state from an old state using a `reducer`, but it means that a mutable reference to the old state must be kept.

A naiive implementation would look something like this: 
``` Kotlin
private var state: State // mutable state

fun handleAction() {
    useCase.loadData()
        // map to a reducer describing changes to the state with the new data
        .map { newData ->
            Reducer<State> { 
                it.copy(data = newData)
            }
        }
        .subscribe { reducer ->
            state = reducer(state) // generate new state with reducer
            _liveData.postValue(state) // emit new state for new observer
        }
}
```

In order to avoid mutable state and to contain the flow of data within the stream, the [`scan`](http://reactivex.io/documentation/operators/scan.html) operator can be used. 

```
The Scan operator applies a function to the first item emitted by the source Observable and then emits the result of that function as its own first emission. It also feeds the result of the function back into the function along with the second item emitted by the source Observable in order to generate its second emission. It continues to feed back its own subsequent emissions along with the subsequent emissions from the source Observable in order to create the rest of its sequence.

This sort of operator is sometimes called an “accumulator” in other contexts.
```

With the `scan` operator providing the previous result for each emission, there is now no need for a mutable state outside of the stream, and the data flow can be kept entirely within the stream.

---

# Conclusion

The final implementation, with `State`, `Event` and `Action`:

``` Kotlin
private val _state = MutableLiveData<State>()
private val _events = UnicastWorkSubject.create<Event>()
private val _actions = PublishSubject.create<Action>()


// route all actions to a subject to make it into a stream of actions
fun handleAction(action: Action) = _actions.onNext(action)

init {
    // define reducers that describe changes to apply to the state
    // and also perform side effects

    val loadDataActionReducer = _actions
            .ofType(Action.LoadData::class.java)
            .flatMap {
                useCase.loadData()
                    // perform side effects that do not affect the state
                    // doOnX operators are perfect for side effects
                    .doOnError {
                        _events.onNext(Event.LoadDataFailure)
                    }
                    // reducer describing changes to the state with new data
                    .map { newData ->
                        Reducer<State> { 
                            it.copy(data = newData, isLoading = false)
                        }
                    }
                    .onErrorReturnItem(
                        Reducer<State> {
                            it.copy(isLoading = false)
                        }
                    )
                    .startWith(
                        Reducer<State> {
                            it.copy(isLoading = true)
                        }
                    )
            }

    val otherActionsReducer = _actions
            .ofType(Action.OtherAction::class.java)
            .flatMap {
                // map to stream of reducer
            }

    // start the stream of actions -> reducer -> state

    Observable
        // 1. create a stream of reducers
        .merge(loadDataActionReducer,otherActionsReducer)
        // 2. use scan to maintain the state stream starting with an initial state
        // [state] is the previous state provided by the scan operator
        // [reducer] is the item emitted by upstream (stream of reducers in step 1)
        .scan(State()) { state, reducer -> reducer(state) }
        // do not have to emit again if the state is unchanged
        .distinctUntilChanged()
        // 3. emit the resulting state to observer
        .subscribe { state ->
            _liveData.postValue(state)
        }
}
```
