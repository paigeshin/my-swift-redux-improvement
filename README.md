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
