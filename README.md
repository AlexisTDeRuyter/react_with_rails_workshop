# SD React With Rails Workshop

## Learning Competencies

* Webpacker Gem
* Pack tags
* How React and Rails work together
* How components will fetch their data 

## Summary
This workshop will build on the comment section created in our previous [intro to React workshop](https://github.com/Devbootcamp/sd-intro-to-react-workshop). We'll be taking the comment section, adding it to a Ruby on Rails app and adding database persistence for the comments. However if you didn't attend that workshop, don't worry! It can function as a stand alone as well. 
- How to generate rails app with webpack flag for react

## Releases

### Release 0:
This workshop isn't about creating a rails app so we've already set up the back end for you. However, Rails 5.1 has some big improvements to the way it handles javascript which are worth noting

Rails 5.1 comes preloaded with the webpacker gem. When we create a project we can add a `webpack` flag and provide an argument for the JS library we want for our front end. Currently React, Angular, Vue, and Elm are supported. So creating this project with React ready to go was as easy as `rails new sd-react-with-rails-workshop -T --database=postgresql --webpack=react`

Now when we view our project we have an `app/javascript/packs` directory. Everything in this packs directory will be compiled by webpack separately from the asset pipeline. This is where we'll put our React code. 

This workshop isn't about creating a rails app so the back end is already set up. We have a Comment model with RESTful routes to return allcomments, create a comment, and delete a comment. 

One note is that along with booting up the rails app with the `rails server` we will have to run `./bin/webpack-dev-server` in a separate terminal window so Webpack will compile our Javascript.

### Release 1: Adding The Comment Section
Ok let's get started! The first step will be taking the comment section we created in the last workshop and adding it to the rails app. In the `/javascript` directory let's add a new directory called `comments`. Inside the comments directory we can copy in the contents of our `/components` directory from the Intro To React project. 

This will give us a directory `/javascript/comments` that contains the files 
- CommentSection.js
- Comments.js
- Comment.js
- Form.js

We just need to make a couple of tweaks and this will work in our Rails project just like it did in our Create React App project. That's the beauty of componentization!

In the `/packs` directory let's create a file called `comments.js` and in that file require the comments folder we just created.
``` JavaScript
require('comments')
```
When we include a folder in the packs directory, it will look for a file called `index` in that folder to render. Let's create an `index.js`. This file's only responsibility will be to render the `CommentSection` component so let's do that!

``` JavaScript
import React, { Component } from 'react';
import CommentSection from './CommentSection'

class CommentIndex extends Component {
  render() {
    <CommentSection />
  }
}

class CommentIndex extends Component {
  render() {
    return (
      <CommentSection />  
    )
  }
}
```

One more thing we need to add in our `index.js`. This component will need to render itself onto the page. 

We can start by importing 'ReactDOM' in our imports at the top of the file.
``` JavaScript
import ReactDOM from 'react-dom'
```
and then adding this bit of code to the bottom of the file
``` JavaScript
document.addEventListener('DOMContentLoaded', () => {
  ReactDOM.render(
    <CommentSection name="CommentSection" />,
    document.body.appendChild(document.createElement('div')),
  )
})
```
This is just creating a div on the page and rendering our CommentIndex component inside it.

Now that our comment section is getting compiled by webpack we just need to include it in a view. We do this using a `pack` tag. In our `/views/comments/index.html.erb` let's add the pack tag `<%= javascript_pack_tag 'comment' %>`. And that's it! If we run the rails server and the webpack dev server we can visit `localhost:3000/comments` and our comment section should work just like it did in our Intro To React project. 

Let's double check and then move on to the next section. As a reminder we'll need to run the rails server in one terminal window `rails server` and the webpack dev server in another `./bin/webpack-dev-server`

### Release 2: Loading Data From The Server
Right now we're just loading comments from an array which doesn't seem particularly useful for anything more than an example. Let's get some data from the database!

In the constructor function for our CommentSection component we're loading the comments on line 12 with 
``` JavaScript
comments: commentList() 
```
We need to start by setting the initial comments to an empty array. 
``` JavaScript
comments: []
```
This is because on load the call to the database will not have returned yet with our comments and the Comments component is expecting an array to map over.

To make the database call we'll be using the [`Fetch`](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) web API. This is native implementation but only has modern broswer support so if you are using it in a production app you will need to use a polyfill.

We'll start by adding an `api` folder in the `javascript` directory with a `CommentApi.js` file inside. The api will be responsible for interacting with the database rather than the component itself.

In the CommentApi add a function to fetch comments from the index. Since we're using RESTful routing conventions this will just be `GET /comments`

This is reallly easy with the Fetch API. It does GET requests by default and we just need to pass it the url we want. 
``` JavaScript
export function getComments() {
  return (
    fetch('/comments.json')
  )
};
```
This will hit the endpoint `/comments.json` and return a promise that we need to handle.

Because we're getting a JSON response we need to call `.json()` on the response. This will return a promise with a parsed response object that we can use to set the state.

``` JavaScript
export function getComments() {
  fetch('/comments.json')
  .then((response) => {
    return response.json();
  }) 
  .then((responseObject) => {
    this.setState({
      comments: responseObject
    }) 
  });
};
```
Now we can use the React lifecycle to send a request the first time the component loads. Let's import the `getComments` function we created in the CommentApi.

``` JavaScript
import { getComments } from '../api/CommentApi'
```
And then call it when the component loads. We need to bind the context of the component to the getComments function so we can set the state correctly inside.

``` JavaScript
componentDidMount() {
  getComments.bind(this)();
}
```

Now we're loading comments from the database!
### Release 3: Sending Data To The Server
We're now getting our comments from the database, but if we create a new comment it doesn't persist. Let's fix that!

We need to add a createComment function to our CommentApi. We can use fetch again but we need to add a couple of things. Fetch can take an options hash as a second argument and we'll add everything there.

We'll need to pass in four options
- the HTTP method 
- the body 
- headers (the CSRF token, the data type we're sending and accepting)
- credentials

The HTTP method is easy, it's going to be a post. We'll get the data for the body from the form and pass it in as an argument. 

For the headers we'll accept 'application/json' and let the server know we're sending 'application/json'. Since we're using a Rails view the CSRF token won't be difficult to get, we can just grab it from the document head.
``` JavaScript
let token = document.head.querySelector("[name=csrf-token]").content;
```
And the we just need to let it know that our request is 'same-origin'.

So our POST request will look like this:
``` JavaScript
fetch('/comments', {
  method: 'post',
  body: newComment,
  headers: {
    'X-CSRF-Token': token,
    'accept': "application/json",
    'Content-Type': 'application/json'
  },
  credentials: 'same-origin'
})
```

Again we'll need to parse the JSON and set the state with the response data (we also want to handle hiding the form when we set the state here). The whole function in our CommentApi will look like this

``` JavaScript
export function createComment(newComment) {
  let token = document.head.querySelector("[name=csrf-token]").content;

  fetch('/comments', {
    method: 'post',
    body: {comment: newComment},
    headers: {
      'X-CSRF-Token': token,
      'accept': "application/json",
      'Content-Type': 'application/json'
    },
    credentials: 'same-origin'
  })
  .then((response) => {
    response.json().then((data) => {
      let list = [...this.state.comments, data]
      this.setState({ 
        showForm: false,
        comments: list
      })
    })
  })
}

```
In our CommentSection component we need to import this function as well
``` JavaScript
import { getComments, createComment } from '../api/CommentApi'
```
And then use it in our handleFormSubmit function. Remember we need to bind the context of our CommentSection component to the form so we can correctly set the state!

``` JavaScript
handleFormSubmit(event) {
  event.preventDefault();
  let newComment = getFormValues(event.target)
  if(newComment.body){
    createComment.bind(this)(newComment)
  }
}
```
#### Release 4: Deleting A Comment

Using the things we've learned to add a delete button
