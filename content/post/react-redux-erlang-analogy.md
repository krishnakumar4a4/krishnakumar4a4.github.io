+++
authors = [
    "Krishna Kumar Thokala",
]
title = "React-Redux and Erlang — A Simple analogy"
description = "React-Redux and Erlang — A Simple analogy"
date = 2017-05-01T09:43:12+05:30
tags = [
    "react",
    "erlang"
]
categories = [
    "react",
    "erlang"
]
series = ["react", "erlang"]
aliases = ["react", "erlang"]
images = ["/images/react-redux-erlang-analogy.png"]
+++

If you know
- Erlang and wanted to know how easy is react-redux to learn, Start reading from “I am an Erlang developer and wanted to know about react-redux”
- React-Redux and wanted to dive into erlang world for greater good, start from erlang synopsis section below.

#### I am an Erlang developer and wanted to know about react-redux:
If you had some experience in using gen_servers and gen_event behaviors? Learning react-redux would be simple for you too, just like me.

In short,

**React** is a UI technology that uses virtual DOM to render pages instantly. **Redux** is a javascript library that gives react the ability to have a centralized store for managing the user state and data.

Any react container can do two things, one is to add/edit the store with actions and the other is to get store updates by subscription.

Redux’s Centralized store/state is similar to gen_server doing the state management.

##### A sample react-redux application will look like below:
**InitialState** in redux is a simple json object that stores the default values to be loaded into the global state/redux store as and when the app is mounted.
> In erlang, it is like the state **record #{}** we define inside a gen_server behavior file.

```js
const initialState = {
      userName: undefined
}
```

----

**Reducers** contain switch statements with each case holding a logic to update the state differently. For every redux action triggered, based on the type of action, it pattern matches to one of the switch cases below and updates the state accordingly.

> In Erlang, these are like set of handle_call/handle_cast callbacks in gen_server.

> For every gen_server:call/cast, a corresponding **handle_call/handle_cast** callback is called to update the state accordingly.

**reducer.js**
```js
function reducer(state = initialState, action) {
  switch (action.type) {
  case GET_USER_NAME:
    return fromJS(state)
            .updateIn(['userName'],() => action.response)
            .toJS()
  default:
    return state
  }
}
```
----

**combineReducers** method merges many reducers.

> In Erlang, it would be like **putting many handle_call/handle_cast callbacks** from different files together in gen_server module.

**createStore** method initializes the redux store by loading the intialState object into the global state/redux store and tags it to the reducers. Render method renders the <HelloWorldApp> react component.

> In Erlang, it should be like the init function being called when the erlang gen_server process is started with gen_server:start/start_link functions. The state record defined in the file will be loaded into process state using the init callback

```js
const rootReducer = combineReducers({
    reducer
})
const store = createStore(rootReducer);
export default class Root extends Component {
  render() {
    return (
      <Provider store={store}>
        <HelloWorldApp />
      </Provider>
    )
  }
}
```
----

HelloWorld react component when mounted, prints default Hello World message on to the webpage along with a Click Here hyperlink. When someone clicks on it, it triggers a **redux action** called getWelcomeName.

> In Erlang, it is similar to **gen_server:call/cast** which is pattern matched against handle_call/handle_cast callbacks.

**mapStateToProps** is the method which allows the react component to subscribe to the store updates. It means, when the state is updated, the new state is mapped to the props of this react component. React component re-renders if any props change occur and hence prints the Welcome <userName> string with userName mapped from the global state/redux store.

In Erlang, as such there is no direct functionality that exists to send events when the state is updated, but it can be assumed to be presence of an event call like gen_event:notify with the updated state at the end of handle_call/handle_cast callback execution. All the events should be sent to an event manager started by gen_event:start API. mapStateProps can be assumed as **gen_event:add_handler** which subscribes to the notifications from the event manager.

**HelloWorldApp.js**

```js
class HelloWorldApp extends Component {
getUserName(){
   this.props.getUserName()
}
render(){
const userName = this.props.userName;
return(
       <div>
         <h1>Hello World</h1>
         <span onClick={this.getUserName.bind(this)}>Click Here</span>
         { userName !== undefined ? 
                           <span> Welcome {userName} </span> :
                           "" }
</div>
);
}
function mapStateToProps(state) {
  return {
    user: state.userName
  }
}
export default connect(mapStateToProps)(AsyncApp)
HelloWorldApp.propTypes = {
  userName: PropTypes.object.isRequired,
  getuserName: PropTypes.func.isRequired
}
```
----

When the action **getUserName** is called, it executes the below method which in turn calls an web endpoint using the fetch method and dispatches the response to a reducer. The dispatched object to the reducer would look like **{type: GET_USER_NAME,response: “<Response obtained>”}**. The value associated with type is used as a pattern match to the switch case present in the reducers. Upon matching, the state is updated with the response.

> In Erlang, this logic can either be put in the handle_call/cast function body or this can be put before the call to **gen_server:call/cast** with record formed from the response. Record would look like **#state{type:GET_USER_NAME, response: “Krishna”}**. Here type can be used to pattern_match a handle_call/cast.

**action.js**

```js
import fetch from 'isomorphic-fetch'
export const GET_USER_NAME = 'GET_USER_NAME'

export function getUserName(dispatch) {
return dispatch => {
    return fetch(`https://<SomeEndPoint>`)
            .then(respone =>
                      dispatch({
                              type: GET_USER_NAME,
                              response
                               })
                  );
    }
}
```
----

Magic happens once the state/store is updated, All the subscribed react components will get new props from the new state and re-renders. React is very good at re-rendering only the part of the DOM that is updated and the rest will remain unchanged. This is an efficient way for rendering webpages.

#### Erlang synopsis:

Let me bring out some synopsis about the erlang’s gen_server and gen_event behavior. If you are quite new, read my blog which explains [why you may choose erlang for one of your next project](https://medium.com/@krishna.thokala2010/know-why-you-may-choose-erlang-7d02e890788d). And also an in depth explanation of all the erlang behaviors with pragmatic examples can be found at [Be on your best Erlang behavior](https://medium.com/@krishna.thokala2010/be-on-your-best-erlang-behavior-f8b478ef14cd).

If you are already familiar enough with erlang, continue with synopsis below.

**gen_server** is a simple state management behavior with handles to update the state either synchronously or asynchronously. It has some API’s and corresponding callbacks.

>gen_server:start_link() → init()  
>gen_server:call() → handle_call()  
>gen_server:cast() → handle_cast()  
>gen_server:stop() → terminate()  

Whereas gen_event is rather different behavior, it has an event manager which acts as sink and consumes every event. It then broadcast events to all the subscribed event handlers.

>gen_event:start_link() → starts a event manager with a name  
>gen_event:add_handler() → Adds a event handler and is subscribed to the event manager, also calls init() function  
>gen_event:notify() → handle_event()  
>gen_event:call() → handle_call()  
>gen_event:stop() → terminate()  

To be more clearer, see the picture below that demostrates the react-redux flow and its erlang equivalents.

{{< figure src="/images/react-redux-erlang-analogy.png">}}

#### References:
Comprehensive Redux tutorial, pre-requisites javascript and react http://redux.js.org/

Basic react-redux app, https://github.com/coryhouse/react-slingshot


