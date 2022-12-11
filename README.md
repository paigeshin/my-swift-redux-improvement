# my-swift-redux-improvement

### Simplifed, Experiment Version 1

```swift
//
//  Store.swift
//  myredux
//
//  Created by paige shin on 2022/12/10.
//

import Combine
import Foundation

enum AppAction {
    case user(UserAction)
}

enum UserAction {
    case set(String)
    case setError(Error)
    case async(AsyncAction)
    
    enum AsyncAction {
        case fetch
        case update(String)
    }
    
}

typealias Middleware = (AppState, AppAction, @escaping(AppAction) -> Void) -> Void
typealias Reducer = (inout AppState, AppAction) -> Void

class AppState {
    @Published var userState: UserState = UserState()
}

class UserState {
    @Published var user: String = "Not Fetched Yet"
    @Published var error: UserError?
}

enum UserError: Error {
    case updateError
}

class Store: ObservableObject {
    
    @Published var appState: AppState
    let reducer: Reducer
    let middlewares: [Middleware]

    init(appState: AppState,
         reducer: @escaping Reducer,
         middlewares: [Middleware]) {
        self.appState = appState
        self.reducer = reducer
        self.middlewares = middlewares
    }
    
    func dispatch(action: AppAction) {
        DispatchQueue.main.async {
            self.reducer(&self.appState, action)
        }
        self.middlewares.forEach { middleware in
            middleware(self.appState, action, self.dispatch)
        }
        
    }
    
}

func appReducer(state: inout AppState, action: AppAction) {
    switch action {
    case .user(let userAction):
        userReducer(state: &state.userState, action: userAction)
    }
}

func userReducer(state: inout UserState, action: UserAction) {
    switch action {
    case .set(let user):
        state.user = user
    default: break
    }
}

func userMiddleware() -> Middleware {
    return { state, action, dispatch in
        switch action {
        case .user(.async(.fetch)):
            DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
                print("Fetched")
                dispatch(.user(.set("Fetched")))
            }
        case .user(.async(.update(let user))):
            print("update user: \(user)")
            dispatch(.user(.setError(UserError.updateError)))
//            DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
//                print("Updated")
//                dispatch(.user(.set(user)))
//            }
        default: break
        }
    }
}

```
### Simplifed, Experiment Version 2
```swift
//
//  Store.swift
//  myredux
//
//  Created by paige shin on 2022/12/10.
//

import Combine
import Foundation

enum AppAction {
    case user(UserAction)
}

enum UserAction {
    case set(String)
    case setError(Error)
    case async(AsyncAction)
    
    enum AsyncAction {
        case fetch
        case update(String)
    }
    
}

struct UserState {
    var user: String = "Not Fetched Yet"
    var error: UserError?
}

enum UserError: Error {
    case updateError
}

class Store: ObservableObject {
    
    @Published var userState: UserState = UserState()
    
    func dispatch(action: AppAction) {
        DispatchQueue.main.async {
            switch action {
            case .user(let userAction):
                userReducer(state: &self.userState, action: userAction)
                switch userAction {
                case .async(let async):
                    userMiddleware()(self.userState, async, self.dispatch(action:))
                default: break
                }
            }
        }
    }
    
}

func userReducer(state: inout UserState, action: UserAction) {
    switch action {
    case .set(let user):
        state.user = user
    default: break
    }
}

func userMiddleware() -> (UserState, UserAction.AsyncAction, @escaping(AppAction) -> Void) -> Void {
    return { state, action, dispatch in
        switch action {
        case .fetch:
            DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
                print("Fetched")
                dispatch(.user(.set("Fetched")))
            }
        case .update(let user):
            print("update user: \(user)")
            dispatch(.user(.setError(UserError.updateError)))
        }
    }
}

```

### Simplifed, Experiment Version 3

```swift
import Foundation

typealias Dispatcher = (ReduxAction) -> Void
typealias Reducer = (inout ReduxState, ReduxAction) -> Void
typealias Middleware = (ReduxState, ReduxAction, @escaping Dispatcher) -> Void

protocol ReduxState { }
protocol ReduxAction { }
protocol ReduxEnvironment { }

final class Store: ObservableObject {

    @Published private(set) var state: ReduxState
    private let reducer: Reducer
    private var middlewares: [Middleware]

    init(reducer: @escaping Reducer,
         state: ReduxState,
         middlewares: [Middleware] = []) {
        self.reducer = reducer
        self.state = state
        self.middlewares = middlewares
    }

    func dispatch(action: ReduxAction) {
        DispatchQueue.main.async { [weak self] in
            guard let strongSelf = self else { return }
            strongSelf.reducer(&strongSelf.state, action)
        }

        // run all middlewares
        self.middlewares.forEach { [weak self] middleware in
            guard let strongSelf = self else { return }
            middleware(strongSelf.state, action, strongSelf.dispatch)
        }
    }
    
    func injectMiddlewars(_ middlewares: [Middleware]) {
        self.middlewares = middlewares
    }

}

func appReducer(state: inout ReduxState, action: ReduxAction) {
//    switch action {
//    case .user(let userAction):
//        userReducer(state: &state.userState, action: userAction)
//    }
}

enum AppAction: ReduxAction {
    case user(action: UserAction)
}

enum UserAction: ReduxAction {
    case fetch
    case fetchComplete(users: [NetworkUser])
    case fetchError(error: Error?)
}

struct AppState: ReduxState {
    var routeState: RouteState = RouteState()
    var authState: AuthState = AuthState()
}

func logMiddleware() -> Middleware {
    return { state, action, dispatch in
        Log.info("⭐️ACTION RECEIVED⭐️: \(action)")
    }
}


```

### Implemented in real project 

```swift 
import Foundation

typealias Dispatcher<Action: ReduxAction> = (Action) -> Void
typealias Reducer<State: ReduxState, Action: ReduxAction> = (_ state: State, _ action: Action) -> State
typealias Middleware<State: ReduxState, Action: ReduxAction> = (State, Action, @escaping Dispatcher<Action>) -> Void

protocol ReduxState: Equatable, Hashable { }
protocol ReduxAction { }
protocol ReduxEnvironment { }

final class Store<State: ReduxState, Action: ReduxAction>: ObservableObject {

    @Published var state: State
    private let reducer: Reducer<State, Action>
    private var middlewares: [Middleware<State, Action>]

    init(state: State,
         reducer: @escaping Reducer<State, Action>,
         middlewares: [Middleware<State, Action>] = []) {
        self.reducer = reducer
        self.state = state
        self.middlewares = middlewares
    }

    func dispatch(action: Action) {
        DispatchQueue.main.async { [weak self] in
            guard let strongSelf = self else { return }
            strongSelf.state = strongSelf.reducer(strongSelf.state, action)
        }

        // run all middlewares
        self.middlewares.forEach { middleware in
            middleware(state, action, dispatch)
        }
        
    }
    
    func inject(middlewares: [Middleware<State, Action>]) {
        self.middlewares = middlewares
    }

}

enum Route {
    case splash
}

struct AppState: ReduxState {
    var route: Route = .splash
    var authState: AuthState = AuthState()
}

import Foundation

enum AppAction: ReduxAction {
    case navigate(Route)
}

func appReducer(_ state: AppState, _ action: ReduxAction) -> AppState {
    var state: AppState = state
    
    return state
}

func logMiddleware() -> Middleware<AppState, AppAction> {
    return { state, action, dispatch in
        Log.info("⭐️ACTION RECEIVED⭐️: \(action)")
    }
}

enum AuthAction: ReduxAction {
    case requestAppleSignIn(ASAuthorizationAppleIDRequest)
    
    enum Async {
        case handleAignSignInRequestResult(Result<ASAuthorization, Error>)
    }
    
}

``
