---
layout: post
title: How to upload files using React and Rails like a boss
date: 2017-09-17
comments: true
categories: [rails, react, upload]
tags: [rails, react, upload]
excerpt_separator: <!-- more -->
---

Handle file upload in React is not a trivial topic, especially with multiple files upload. In this article, I would like to go in depth how we can make use of React, FormData to upload to Rails smoothly. The features that we are going to build in this article will include

- Single/Multiple File Upload.
- Accept certain file types.
- Show progress bar when files are being uploaded
- Preview the files before they are uploaded
- How Rails can handle to multiple file uploads

<!-- more -->

This is the screenshot of what we will have after finished building;

<p style='text-align:center;' markdown='1'><img src='/public/images/multiple-files-upload.jpg' style='display:inline;'/></p>

## 1. Rails App

I would use ``Paperclip`` as the gem to handle file upload in Rails. However, you are freely to choose any file upload handler in Rails.

As usual, first start a Rails project:

```bash
rails new files-upload-demo --database=mysql --api
```

Next, add the ``paperclip`` gem:

```rb
gem 'paperclip', '~> 5.1.0'
```

Then run:

```bash
bundle install
```

Next, generate a model called ``book`` and a model called ``cover``. A book will have many covers:

```bash
rails g model Book title description:text
rails g model Cover book_id:integer
rails g paperclip covers photo
```

Paperclip will generate a migration without a version which will cause an issue when we run the migration later on. Let's fix it by adding ``[5.1]`` after ``ActiveRecord::Migration``

```ruby
# db/migrate/20170917030652_add_attachment_photo_to_covers.rb
class AddAttachmentPhotoToCovers < ActiveRecord::Migration[5.1]
# ...
end
```

After that, we will run the following commands to create and migrate database:

```bash
bundle exec rake db:create
bundle exec rake db:migrate
```

Let's generate books scaffold with skip flag to skip through the files that we have already generated

```bash
rails g scaffold books --skip
```

After that, we would need to modify the function ``book_params`` to allow approriate parameters

```ruby
# app/controllers/books_controller
class BooksController < ApplicationController
  # ...
  def book_params
    params.require(:book).permit(
      [
        :title,
        :description,
        covers_attributes: %I[
          id
          photo
          _destroy
        ]
      ]
    )
  end
end
```

We also need ``rack-cors`` gem to let the request call from cross domain. Add this line into Gemfile

```ruby
# Gemfile
gem 'rack-cors'
```

and then run

```bash
bundle install
```

Then set up to allow the whilelisted domain to send request across. Add the following code to ``application.rb``:

```ruby
# config/application.rb
config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'
    resource '*',
             headers: :any,
             methods: %I[get post options delete patch puts]
  end
end
```

Note that, you will need to adjust the ``origins`` and ``resources`` in the above code to prevent any unexpected requests from other domains

Now, inside ``book.rb`` model, we will need to add the following code to declare the relationship and some basic validations:

```ruby
class Book < ApplicationRecord
  validates :title, presence: true
  validates :description, presence: true

  has_many :covers, dependent: :destroy
  accepts_nested_attributes_for :covers, allow_destroy: true

  def as_json(_opts = {})
    {
      id: id,
      title: title,
      description: description,
      errors: errors,
      cover_photos: covers.map do |x|
        {
          url: x.photo.url.absolute_url,
          name: x.photo_file_name,
          id: x.id
        }
      end
    }
  end
end
```

The ``as_json`` is used whenever a ``render json: Book.all`` or things similar to that. Note that, we return back the ``errrors`` and the ``cover_photos`` so that it can be rendered at the front end.

We would also need to add some code into ``cover.rb`` like following:

```ruby
class Cover < ApplicationRecord
  belongs_to :book, inverse_of: :covers
  has_attached_file \
    :photo,
    styles: { thumb: ['32x32#', 'jpg'] },
    convert_options: {
      all: '-interlace Plane'
    },
    default_url: '/images/default_:style_photo.png'

  validates_attachment_presence :photo
  validates_attachment_file_name :photo, matches: [/png\Z/, /jpe?g\Z/, /gif\Z/]
end
```

We are done with Rails app for now. Let's go to the react app.

## 2. React App

Let install ``create-react-app`` in order to create a react app easily

```bash
npm install -g create-react-app
```

Then create a React app by issuing a command:

```bash
create-react-app files-upload-demo-react
```

Next, let's bring in React Router so that we can have multiple URLs for the Listing, New, Edit function

Let's move into the files-upload-demo-react folder:

```bash
cd files-upload-demo-react
```

Then

```bash
yarn add react-router-dom react-router
```

Let's also add ``axios`` to make ajax calls:

```bash
yarn add axios
```

Then create a file called ``Routes.js`` inside ``src`` folder:

```js
// src/Routes.js

import React from 'react';
import { Switch, HashRouter as Router, Route } from 'react-router-dom';
import createBrowserHistory from 'history/createBrowserHistory';

import BookIndex from './Book/Index';
import BookNew from './Book/New';
import BookEdit from './Book/Edit';
import NotFound from './NotFound';

const history = createBrowserHistory();
const Routes = () =>
  <Router history={history}>
    <Switch>
      <Route path="/books/:id/edit" component={BookEdit} />
      <Route path="/books/new" component={BookNew} />
      <Route path="/books/" component={BookIndex} />
      <Route path="*" component={NotFound} />
    </Switch>
  </Router>;

export default Routes;
```

These are pretty much React Route functions. In this case, we have mapped the followings:

- ``books/:id/edit`` to ``BookEdit`` component
- ``books/new`` to ``BookNew`` component
- ``books/`` to ``BookIndex`` component

We would also need to create an ``axiosClient`` to ensure that the API base URL is pointing to the Rails server. To do that, create a file ``src/axiosClient.js`` with the following content:

```js
let axios = require('axios');

let axiosClient = axios.create({
  baseURL: 'http://localhost:3000'
});

export default axiosClient;
```

I also want this React Project to run on port ``8000`` instead of the default port ``3000``. To do that, we need to create ``.env`` and add in the line:

```
PORT=8000
```

We would need to change the root ``index.js`` to make use of the Router:

```js
// src/index.js
import React from 'react';
import ReactDOM from 'react-dom';
import Routes from './Routes';
import registerServiceWorker from './registerServiceWorker';

ReactDOM.render(<Routes />, document.getElementById('root'));
registerServiceWorker();
```

Now let's move on to create ``Book/Index`` folder inside ``src``:

```bash
mkdir -p src/Book/Index
```

Next, let's create a file to list down all the available books.
To do that, create the file ``src/Book/Index/index.js`` with the following content:

```js
import React, { Component } from 'react';
import axiosClient from '../../axiosClient';

class BookIndex extends Component {
  constructor(props) {
    super(props);
    this.state = { books: [] };
  }

  componentWillMount() {
    axiosClient.get('/books.json').then(response => {
      this.setState({ books: response.data });
    });
  }

  render() {
    return (
      <div className="BookIndex col-md-12" style={{ marginTop: 10 }}>
        <div className="clearfix">
          <div className="pull-right">
            <button
              onClick={e => this.handleNewBook()}
              className="btn btn-success">
              New Book
            </button>
          </div>
        </div>
        <table className="table">
          <thead>
            <tr>
              <th>#</th>
              <th>Title</th>
              <th>Description</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {this.renderTableBody()}
          </tbody>
        </table>
      </div>
    );
  }

  handleNewBook() {
    this.props.history.push('/books/new');
  }

  renderTableBody() {
    return this.state.books.map(book => {
      return (
        <tr key={book.id}>
          <td>
            {book.id}
          </td>
          <td>
            {book.title}
          </td>
          <td>
            {book.description}
          </td>
          <td>
            <button
              onClick={e => this.handleEdit(book.id)}
              className="btn btn-primary">
              Edit
            </button>
            &nbsp;
            <button
              onClick={e => this.handleRemove(book.id)}
              className="btn btn-danger">
              Remove
            </button>
          </td>
        </tr>
      );
    });
  }

  handleEdit(bookId) {
    this.props.history.push(`/books/${bookId}/edit`);
  }

  handleRemove(bookId) {
    let books = this.state.books;
    books = books.filter(book => {
      return book.id !== bookId;
    });
    this.setState({ books: books });
    axiosClient.delete(`/books/${bookId}`);
  }
}

export default BookIndex;
```

This ``index.js`` file is used to render the list of books and call the API ``/books.json`` to get the data from server to display. It also handles the edit and remove action in ``handleEdit`` and ``handleRemove`` callback.

Moving forwards, let's create ``Book/Form`` folder. This component will be shared between edit and create book component:

```bash
mkdir src/Book/Form
```

Then create the file ``src/Book/Form/index.js`` with the following content:

```js
import React, { Component } from 'react';
import axiosClient from '../axiosClient';
import './Index.css';

class BookForm extends Component {
  render() {
    return (
      <div className="BookForm">
        <form>
          <button
            onClick={e => this.handleFormSubmit()}
            className="btn btn-primary">
            Save
          </button>
          &nbsp;
          <button
            onClick={e => this.handleCancel()}
            className="btn btn-default">
            Cancel
          </button>
        </form>
      </div>
    );
  }

  handleCancel() {}
  handleFormSubmit() {}
}

export default BookForm;
```

We start with creating a simple form view for BookForm. This file will eventually house the logic for both update and create a Book.

Moving forwards, let's create ``Book/New`` folder

```bash
mkdir src/Book/New
```

And create an index file ``src/Book/New/index.js`` with the following content:

```js
// src/Book/New/index.js

import React, { Component } from 'react';
import BookForm from '../Form';

class BookNew extends Component {
  render() {
    return (
      <div className="BookNew col-md-8 col-md-offset-2">
        <h2>New Book</h2>
        <BookForm history={this.props.history} match={this.props.match} />
      </div>
    );
  }
}

export default BookNew;
```

and also create ``BookEdit`` folder

```bash
mkdir src/Book/Edit
```

And create an index file ``src/BookEdit/index.js`` with the following content:

```js
// src/Book/Edit/index.js

import React, { Component } from 'react';
import BookForm from '../Form';

class BookEdit extends Component {
  render() {
    return (
      <div className="BookEdit col-md-8 col-md-offset-2">
        <h2>Edit Book</h2>
        <BookForm history={this.props.history} match={this.props.match} />
      </div>
    );
  }
}

export default BookEdit;
```

As you can see, both ``BookEdit`` and ``BookNew`` component are using ``BookForm`` component to handle the form. The only difference in ``BookForm`` would be how we handle based on the URL. If the URL has the ``id`` params, ``BookForm`` component would need to pull the content of the Book first to render out into the form.

To start, let's create some initial state for ``BookForm`` like below:


```js
//...
state = {
  selectedBookCoverFiles: [],
  submitFormProgress: 0,
  isSubmittingForm: false,
  didFormSubmissionComplete: false,
  book: {
    id: this.props.match.params.id,
    title: '',
    description: '',
    errors: {}
  }
};
// ...
```

As mentioned ealier, for edit function, a ``id`` will be passed over to the URL. By using this ``id``, we can pull the content of the book in ``componentWillMount`` like below:

```js
// ...
componentWillMount() {
  if (this.props.match.params.id) {
    axiosClient.get(`/books/${this.props.match.params.id}`).then(response => {
      this.setState({
        selectedBookCoverFiles: response.data.cover_photos,
        book: {
          id: response.data.id,
          title: response.data.title,
          description: response.data.description,
          errors: {}
        }
      });
    });
  }
}
// ...
```

Next, let's update the render function in this component to render out the form:

```js
// ...
render() {
  return (
    <div className="BookForm">
      <form>

        <div className="form-group">
          <label>Title</label>
          <input
            type="text"
            onChange={e => this.handleBookTitleChange(e)}
            value={this.state.book.title}
            className="form-control"
          />
          {this.renderBookTitleInlineError()}
        </div>

        <div className="form-group">
          <label>Description</label>
          <textarea
            type="text"
            onChange={e => this.handleBookDescriptionChange(e)}
            value={this.state.book.description}
            className="form-control"
          />
          {this.renderBookDescriptionInlineError()}
        </div>

        <div className="form-group">
          <label>Covers</label>
          {this.renderUploadCoversButton()}
          {this.renderSelectedBookCoverFiles()}
        </div>

        {this.renderUploadFormProgress()}

        <button
          disabled={this.state.isSubmittingForm}
          onClick={e => this.handleFormSubmit()}
          className="btn btn-primary">
          {this.state.isSubmittingForm ? 'Saving...' : 'Save'}
        </button>
        &nbsp;
        <button
          disabled={this.state.isSubmittingForm}
          onClick={e => this.handleCancel()}
          className="btn btn-default">
          Cancel
        </button>

      </form>
      <br />
    </div>
  );
}
// ...
```

We would need to put up these functions below to handle the change event on title, description and handle the inline error displays:

```js
// ...
handleBookTitleChange(e) {
  let { book } = this.state;
  book.title = e.target.value;
  this.setState({ book: book });
}

handleBookDescriptionChange(e) {
  let { book } = this.state;
  book.description = e.target.value;
  this.setState({ book: book });
}

renderBookTitleInlineError() {
  if (this.state.book.errors.title) {
    return (
      <div className="inline-error alert alert-danger">
        {this.state.book.errors.title.join(', ')}
      </div>
    );
  } else {
    return null;
  }
}

renderBookDescriptionInlineError() {
  if (this.state.book.errors.description) {
    return (
      <div className="inline-error alert alert-danger">
        {this.state.book.errors.description.join(', ')}
      </div>
    );
  } else {
    return null;
  }
}
// ...
```

The core of this article is the function to render out the upload covers button as shown below:

```js
// ...
getNumberOfSelectedFiles() {
  return this.state.selectedBookCoverFiles.filter(el => {
    return el._destroy !== true;
  }).length;
}

renderUploadCoversButton() {
  let numberOfSelectedCovers = this.getNumberOfSelectedFiles();
  return (
    <div>
      <input
        name="covers[]"
        ref={field => (this.bookCoversField = field)}
        type="file"
        disabled={this.state.isSubmittingForm}
        multiple={true}
        accept="image/*"
        style={{
          width: 0.1,
          height: 0.1,
          opacity: 0,
          overflow: 'hidden',
          position: 'absolute',
          zIndex: -1
        }}
        id="book_covers"
        onChange={e => this.handleBookCoversChange(e)}
        className="form-control"
      />
      <label
        disabled={this.state.isSubmittingForm}
        className="btn btn-success"
        htmlFor="book_covers">
        <span className="glyphicon glyphicon-cloud-upload" />
        &nbsp; &nbsp;
        {numberOfSelectedCovers === 0
          ? 'Upload Files'
          : `${numberOfSelectedCovers} file${numberOfSelectedCovers !== 1
              ? 's'
              : ''} selected`}
      </label>
    </div>
  );
}

handleBookCoversChange() {
  let selectedFiles = this.bookCoversField.files;
  let { selectedBookCoverFiles } = this.state;
  for (let i = 0; i < selectedFiles.length; i++) {
    selectedBookCoverFiles.push(selectedFiles.item(i));
  } //end for

  this.setState(
    {
      selectedBookCoverFiles: selectedBookCoverFiles
    },
    () => {
      this.bookCoversField.value = null;
    }
  );
}
// ...
```

Few things to be noted in the function above:

1. - We use the following inline style to hide the upload button. This is useful if you want to hide the file yet still want it to be submitted when the form is submitted.

```js
{
  width: 0.1,
  height: 0.1,
  opacity: 0,
  overflow: 'hidden',
  position: 'absolute',
  zIndex: -1
}
```

2. - The label has the HTML attribute ``for`` matched with the file input id ``book_covers``. This is helpful as user can click on the label to trigger file selection, which is the same effect as clicking on the file input but without javascript.

3. - ``handleBookCoversChange`` will inject into the existing ``this.state.selectedBookCoverFiles`` the list of files selected by user.

Next would be the function to render what user has chosen to upload:

```js
// ...
renderSelectedBookCoverFiles() {
  let fileDOMs = this.state.selectedBookCoverFiles.map((el, index) => {
    if (el._destroy) { // we use _destroy to mark the removed photo
      return null;
    }

    return (
      <li key={index}>
        <div className="photo">
          <img
            width={150}
            src={el.id ? el.url : URL.createObjectURL(el)}
            style={{ alignSelf: 'center' }}
          />
          <div
            className="remove"
            onClick={() => this.removeSelectedBookCoverFile(el, index)}>
            <span style={{ top: 2 }} className="glyphicon glyphicon-remove" />
          </div>
        </div>
        <div className="file-name">
          {el.name}
        </div>
      </li>
    );
  });

  return (
    <ul className="selected-covers">
      {fileDOMs}
    </ul>
  );
}
// ...
```

As shown in the above function, we will render out the photo if the photo is marked as ``_destroy``. Otherwise, it would be shown with an image, a cross icon at the top right corner to handle the deletion and the file name.

The function to handle the deletion is shown below:

```js
// ...
removeSelectedBookCoverFile(cover, index) {
  let { selectedBookCoverFiles } = this.state;
  if (cover.id) { // cover file that has been uploaded will be marked as destroy
    selectedBookCoverFiles[index]._destroy = true;
  } else {
    selectedBookCoverFiles.splice(index, 1);
  }

  this.setState({
    selectedBookCoverFiles: selectedBookCoverFiles
  });
}
// ...
```

For those cover photos that has the id field, when it is removed, we would simply mark the field ``_destroy`` as true and move on. Otherwise, we would splice the array to completely remove that element. Resetting state would handle all the voiew updating.

The last 2 functions that we will need to put in is the ``handleCancel``:

```js
// ...

handleCancel() {
  this.props.history.push('/books');
}

// ...
```
which is pretty straight forward, we can simply redirect the user back to the books listing if the cancel button got hit.

And the ``handleFormSubmit`` function to handle for both cases: create a new book and update an existing book:


```js
// ...
handleFormSubmit() {
  let { book } = this.state;
  book.errors = {};
  this.setState(
    {
      isSubmittingForm: true,
      book: book
    },
    () => {
      this.submitForm();
    }
  );
}

buildFormData() {
  let formData = new FormData();
  formData.append('book[title]', this.state.book.title);
  formData.append('book[description]', this.state.book.description);

  let { selectedBookCoverFiles } = this.state;
  for (let i = 0; i < selectedBookCoverFiles.length; i++) {
    let file = selectedBookCoverFiles[i];
    if (file.id) {
      if (file._destroy) {
        formData.append(`book[covers_attributes][${i}][id]`, file.id);
        formData.append(`book[covers_attributes][${i}][_destroy]`, '1');
      }
    } else {
      formData.append(
        `book[covers_attributes][${i}][photo]`,
        file,
        file.name
      );
    }
  }
  return formData;
}

submitForm() {
  let submitMethod = this.state.book.id ? 'patch' : 'post';
  let url = this.state.book.id
    ? `/books/${this.state.book.id}.json`
    : '/books.json';

  axiosClient
    [submitMethod](url, this.buildFormData(), {
      onUploadProgress: progressEvent => {
        let percentage = progressEvent.loaded * 100.0 / progressEvent.total;
        this.setState({
          submitFormProgress: percentage
        });
      }
    })
    .then(response => {
      this.setState({
        didFormSubmissionComplete: true
      });
      this.props.history.push('/books');
    })
    .catch(error => {
      let { book } = this.state;
      book.errors = error.response.data;
      this.setState({
        isSubmittingForm: false,
        submitFormProgress: 0,
        book: book
      });
    });
}
// ...
```

Let's dive into this function. First we assign a submit method depend on whether there is a book id. It will be a ``POST`` if there is no book id, otherwise, it will be a ``PATCH``, following RESTful standard defined by Rails.

Next is the URL, for ``create`` and ``update``, the URL will be different. For ``create``, it will be simply ``/books.json``, whereas for update it will be ``/books/<id>.json``.

We then use ``axiosClient`` to call the coresponding method (post/patch) with the form data built in the function ``buildFormData``. This ``buildFormData`` function simply creates an empty form data and then add in the title, description following by the selected files. For files that have been previously uploaded and has the ``_destroy`` mark, the ``_destroy`` mark would be sent to the server as well as its id so that Rails can remove these files internally.

With the submission, there would be 2 cases, success and failure.

For success case, we can simply redirect the user back to the book listing. You could also redirect user to the book view page. However, in this scope of this article, we will keep it simple as what it is.

For the failure case (status code ``422`` - Rails ROLLBACK) we would extract out the response data returned by Rails and reassign it back to the main state so that React can handle the view rendering. This response back from Rails carries the ``errors`` attribute in the ``book``.

We would also need to add in the function to display the progress bar when uploading files:

```js
// ...
renderUploadFormProgress() {
  if (this.state.isSubmittingForm === false) {
    return null;
  }

  return (
    <div className="progress">
      <div
        className={
          'progress-bar progress-bar-info progress-bar-striped' +
          (this.state.submitFormProgress < 100 ? 'active' : '')
        }
        role="progressbar"
        aria-valuenow={this.state.submitFormProgress}
        areaValuemin="0"
        areaValuemax="100"
        style={{ width: this.state.submitFormProgress + '%' }}>
        {this.state.submitFormProgress}% Complete
      </div>
    </div>
  );
}
// ...
```

To make the the style looks nice, I add the style file ``src/Book/Form/Index.css`` with the content below:

```css
div.BookForm div.inline-error {
  padding: 5px;
  border-radius: 0 0 4px 4px;
  z-index: -1;
  top: -2px;
  position: relative;
}

div.BookForm ul.selected-covers {
  flex-wrap: wrap;
  padding: 0;
  list-style-type: none;
  display: flex;
  margin-top: 10px;
}

div.BookForm ul.selected-covers li {
  width: 150px;
  overflow: hidden;
  margin-right: 10px;
  margin-bottom: 10px;
}

div.BookForm ul.selected-covers li div.photo {
  width: 150px;
  height: 150px;
  overflow: hidden;
  background: #000;
  display: flex;
  border-radius: 4px;
  border: 2px solid #333;
  position: relative;
}

div.BookForm ul.selected-covers li div.photo div.remove {
  color: #fff;
  position: absolute;
  top: 10px;
  background: #000;
  border: 2px solid #fff;
  border-radius: 20px;
  padding: 2.5;
  width: 25px;
  height: 25px;
  text-align: center;
  cursor: pointer;
  box-shadow: 0 0 5px #000;
  right: 10px;
}

div.BookForm ul.selected-covers li div.file-name {
  color: #999;
  text-align: center;
  margin-top: 10px;
}
```

and import it at the head of ``BookForm`` component:

```js
// ...

import './Index.css';

// ...
```

I also want to make the form and the listing nice, so I have embedded the bootstrap CSS into the ``<head>`` tag of ``index.html`` in this project:

```html
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">
```

Finally, you can start both Rails and React server and enjoy the show :D

And that's it. You can now start playing with the chat app:

<p style='text-align:center;' markdown='1'>
  <img src='/public/gifs/file-uploads.gif' alt="ReactJS File Upload" style='display:inline;'/>
</p>

The source file for this article is made publicly available on the following URLs:

[https://github.com/jameshuynh/blog-codes/tree/master/files-upload-demo](https://github.com/jameshuynh/blog-codes/tree/master/files-upload-demo)
