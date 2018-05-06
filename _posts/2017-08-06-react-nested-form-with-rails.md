---
layout: post
title: React Nested Form with Rails
date: 2017-08-06
comments: true
categories: [rails, react js, nested form]
tags: [rails, react js, nested form]
excerpt_separator: <!-- more -->
---

We used to build nested form in Rails using Rails helper. However, due to the demand of moving to React JS, we would also love to be able to build nested form in React JS, yet we should still able to utilise all the lovely facility & standard provided by Rails. In this article, I will go through an example on how we build a nested form with Project & Task, i.e. one project can have many tasks and we can keep adding task in the same form as the project.

<!-- more -->

## 1. Create a Rails app with API only

Create a new Rails app with MySQL database (or any database that you prefer) and limit this Rails app to have only API

```bash
rails new rails-nested-form --database=mysql --api
```

Next, create a model called ``project`` with ``name`` attribute:

```bash
rails g model project name
```

Then create a model called ``task`` with ``title`` and ``project_id`` attribute:

```bash
rails g model task title project_id:integer
```

Let's create the database and run the migration:

```bash
rails db:create && rails db:migrate
```

We will need to define the relationship between ``project`` and ``task``:


```ruby
# app/models/project.rb
class Project < ApplicationRecord
  validates :name, presence: true
  has_many :tasks, dependent: :destroy
  accepts_nested_attributes_for :tasks, allow_destroy: true
end
```

and for task model

```ruby
# app/models/task.rb
class Task < ApplicationRecord
  validates :title, presence: true
  belongs_to :project, inverse_of: :tasks
end
```

Next, let's create a controller and action to handle form submission:

```bash
rails g controller projects
```

And let's add in the create and update action. Note that we will handle the edit and new view in the React JS app

```ruby
# app/controllers/projects_controller

#...
def index
  json = Project.all.map do |project|
    {
      id: project.id,
      name: project.name
    }
  end

  render json: json
end

def show
  project = Project.find(params[:id])
  render json: project_json(project)
end

def create
  project = Project.new(project_params)
  result = project.save
  render project_json(project), status: result ? 200 : 422
end

def update
  project = Project.find(params[:id])
  project.attributes = project_params
  result = project.save
  render project_json(project), status: result ? 200 : 422
end

def destroy
  project = Project.find(params[:id])
  project.destroy
  render json: { result: :ok }
end

private

def project_json(project)
  {
    id: project.id,
    name: project.name,
    errors: project.errors,
    tasks: project.tasks.map do |task|
      {
        id: task.id,
        title: task.title,
        errors: task.errors,
        _destroy: task._destroy
      }
    end
  }
end

def project_params
  params
  .require(:project)
  .permit(:name, tasks_attributes: [:title, :_destroy, :id])
end
#...
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

And finally, add an resources routes into ``routes.rb``

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :projects
end
```

We have done with setting up the rails project, let's move on to create the ReactJS app

## 2. Create ReactJS App

Let install ``create-react-app`` in order to create a react app easily

```bash
npm install -g create-react-app
```

Then create a React app by issuing a command:

```bash
create-react-app reactjs-nested-form
```

Next, let's bring in React Router so that we can have multiple URLs for the Listing, New, Edit function


Let's move into the reactjs-nested-form folder:

```bash
cd reactjs-nested-form
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

import IndexProject from './IndexProject';
import NewProject from './NewProject';
import EditProject from './EditProject';
import NotFound from './NotFound';

const history = createBrowserHistory();
const Routes = () =>
  <Router history={history}>
    <Switch>
      <Route path="/projects/:id/edit" component={EditProject} />
      <Route path="/projects/new" component={NewProject} />
      <Route path="/projects/" component={IndexProject} />
      <Route path="*" component={NotFound} />
    </Switch>
  </Router>;

export default Routes;
```

These are pretty much React Route functions. In this case, we have mapped the followings:

- ``projects/:id/edit`` to ``EditProject`` component
- ``projects/new`` to ``NewProject`` component
- ``projects/`` to ``IndexProject`` component

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

Now let's move on to create ``IndexProject`` folder inside ``src``:

```bash
mkdir src/IndexProject
```

Next, let's create a file to list down all the available projects.
To do that, create the file ``src/IndexProject/index.js`` with the following content:

```js
import React, { Component } from 'react';
import axiosClient from '../axiosClient';

class IndexProject extends Component {
  constructor(props) {
    super(props);
    this.state = { projects: [] };
  }

  componentWillMount() {
    axiosClient.get('/projects.json').then(response => {
      this.setState({ projects: response.data });
    });
  }

  render() {
    return (
      <div className="IndexProjesct">
        <table className="table">
          <thead>
            <tr>
              <th>#</th>
              <th>Name</th>
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

  renderTableBody() {
    return this.state.projects.map(project => {
      return (
        <tr key={project.id}>
          <td>
            {project.id}
          </td>
          <td>
            {project.name}
          </td>
          <td>
            <button
              onClick={e => this.handleEdit(project.id)}
              className="btn btn-primary">
              Edit
            </button>
            <button
              onClick={e => this.handleRemove(project.id)}
              className="btn btn-danger">
              Remove
            </button>
          </td>
        </tr>
      );
    });
  }

  handleEdit(projectId) {
    this.props.history.push(`/projects/${projectId}/edit`);
  }

  handleRemove(projectId) {
    let projects = this.state.projects;
    projects = projects.filter(project => {
      return project.id !== projectId;
    });
    this.setState({ projects: projects });
    axiosClient.delete(`/projects/${projectId}`);
  }
}

export default IndexProject;
```

This ``index.js`` file is used to render the list of projects and call the API ``/projects.json`` to get the data from server to display. It also handle the edit and remove action in ``handleEdit`` and ``handleRemove`` callback.

Moving forwards, let's create ``ProjectForm`` folder. This component will be shared between edit and create project component:

```bash
mkdir src/ProjectForm
```

Then create the file ``src/ProjectForm/index.js`` with the following content:

```js
import React, { Component } from 'react';
import axiosClient from '../axiosClient';
import './Index.css';

class ProjectForm extends Component {
  render() {
    return (
      <div className="ProjectForm">
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

export default ProjectForm;
```

We start with creating a simple form view for ProjectForm. This file will eventually house the logic for both update and create a Project.

Moving forwards, let's create ``NewProject`` folder

```bash
mkdir src/NewProject
```

And create an index file ``src/NewProject/index.js`` with the following content:

```js
import React, { Component } from 'react';
import ProjectForm from '../ProjectForm';

class NewProject extends Component {
  render() {
    return (
      <div className="NewProject col-md-8 col-md-offset-2">
        <h2>New Project</h2>
        <ProjectForm
          history={this.props.history}
          match={this.props.match} />
      </div>
    );
  }
}

export default NewProject;
```

and also create ``EditProject`` folder

```bash
mkdir src/EditProject
```

And create an index file ``src/EditProject/index.js`` with the following content:

```js
import React, { Component } from 'react';
import ProjectForm from '../ProjectForm';

class EditProject extends Component {
  render() {
    return (
      <div className="EditProject col-md-8 col-md-offset-2">
        <h2>Edit Project</h2>
        <ProjectForm 
          history={this.props.history} 
          match={this.props.match} />
      </div>
    );
  }
}

export default EditProject;
```

As you can see, both ``EditProject`` and ``NewProject`` component are using ``ProjectForm`` component to handle the form. The only difference in ``ProjectForm`` would be how we handle based on the URL. If the URL has the ``id`` params, ``ProjectForm`` component would need to pull the content of the Project first to render out into the form.

To start, let's create some initial state for ``ProjectForm`` like below:


```js
//...
constructor(props) {
  super(props);
  this.emptyTask = {
    title: '',
    id: null,
    errors: {},
    _destroy: false
  };

  this.state = {
    project: {
      name: '',
      errors: {},
      tasks_attributes: [Object.assign({}, this.emptyTask)]
    }
  };
}
// ...
```

We assign ``emptyTask`` as an instance variable to this object so that we can keep reusing it. We also initialise a state with the initial project attributes. Note that, in this case, we are initialising the project with a predefined empty task so that when we display the form, it will already have a task form there.

Each entity ``project`` and ``task`` will have an attribute named ``errors``. This is to help to keep track of the errors sent back from the server.

As mentioned ealier, for edit function, a ``id`` will be passed over to the URL. By using this ``id``, we can pull the content of the project in ``componentWillMount`` like below:

```js
// ...
componentWillMount() {
  if (this.props.match.params.id) {
    axiosClient
      .get(`/projects/${this.props.match.params.id}`)
      .then(response => {
        this.setState({ project: response.data });
      });
  }
}
// ...
```

Next, let's update the render function in this component to render out the nested form for task:

```js
// ...
render() {
  return (
    <div className="ProjectForm">
      <form>
        <div className="form-group">
          <label>Name</label>
          <input
            type="text"
            onChange={e => this.handleProjectNameChange(e)}
            value={this.state.project.name}
            className="form-control"
          />
          {this.renderProjectNameInlineError()}
        </div>
        <hr />
        <div className="tasks-fieldset">
          <h3>Tasks</h3>
          {this.renderTasksForm()}
          <button
            className="btn btn-success"
            onClick={e => this.handleAddTask()}>
            + Add Task
          </button>
        </div>
        <br />
        <button
          onClick={e => this.handleFormSubmit()}
          className="btn btn-primary">
          Save
        </button>
        &nbsp;
        <button
          onClick={e => this.handleCancel()}
          className="btn btn-default">
          Save
        </button>{' '}
      </form>
    </div>
  );
}
// ...
```

We have just added 2 things. First is:

```js
<div className="form-group">
  <label>Name</label>
  <input
    type="text"
    onChange={e => this.handleProjectNameChange(e)}
    value={this.state.project.name}
    className="form-control"
  />
  {this.renderProjectNameInlineError()}
</div>
```

This is a simple text field, which helps to keep track of the project's name and its change. We would also need to add in the callback for ``handleProjectNameChange`` to keep our state in sync with this field

```js
//...
handleProjectNameChange(e) {
  let project = this.state.project;
  project.name = e.target.value;
  this.setState({ project: this.state.project });
}
//...
```

This function will update the state ``project.name`` and sync it back to the state of this component.

And also implement the function ``renderProjectNameLineError`` like below:

```js
// ...

renderProjectNameInlineError() {
  if (this.state.project.errors.name) {
    return (
      <div className="inline-error alert alert-danger">
        {this.state.project.errors.name.join(', ')}
      </div>
    );
  } else {
    return null;
  }
}

// ...
```

This function is quite straight forward, we will display out the error if there is an error with with the project name.

The second thing that we added to the ``render`` function is:

```js
<div className="tasks-fieldset">
  <h3>Tasks</h3>
  {this.renderTasksForm()}
  <button
    className="btn btn-success"
    onClick={e => this.handleAddTask()}>
    + Add Task
  </button>
</div>
```

This piece of code is trying to render the tasks form so that multiple task forms can be housed under ``tasks-fieldset`` div. Also, there is a button to handle the action of adding new task. To move forwards, let implement the function ``renderTasksForm`` like below:

```js
//...
{% raw %}
renderTasksForm() {
  let counter = 0;
  return this.state.project.tasks_attributes.map((task, index) => {
    if (task._destroy === false) {
      let taskDOM = (
        <div className="task-form" key={index}>
          <div className="form-group">
            <div className="clearfix" style={{ marginBottom: 5 }}>
              <label>
                Task {counter + 1}
              </label>
              <button
                className="btn btn-danger"
                style={{ padding: '5px 10px', float: 'right' }}
                onClick={e => this.handleRemoveTask(task)}>
                X
              </button>
            </div>
            <input
              placeholder="Title"
              onChange={event => this.onTaskTitleChange(event, task)}
              type="text"
              value={task.title}
              className="form-control"
            />
            {this.renderTaskInlineError(task)}
          </div>
        </div>
      );
      counter++;

      return taskDOM;
    } else {
      return null;
    }
  });
}

// ...
{% endraw %}
```

This function loops through the ``task_attributes`` in the main state and render out the task form. Each task form consists of an input field for task's title and a button to help to delete the task. If the task has the field ``_destroy`` set to false, it will not be rendered out. This field would be marked as ``true`` when we hit the delete button on the task.

Let's move on by adding the function ``handleRemoveTask``:

```js
// ...

handleRemoveTask(task) {
  task._destroy = true;
  this.setState({ project: this.state.project });
}

// ...
```

As you can see, we can simply handle this by assign the task's _destroy attribute to true and set its state back so that it can reflect in the view again.

We would also need to have another function ``onTaskTitleChange``, which is almost identical to ``handleRemoveTask``j

```js
// ...

onTaskTitleChange(event, task) {
  task.title = event.target.value;
  this.setState({ project: this.state.project });
}

// ...
```

and another function is ``renderTaskInlineError`` to show the inline error of task's title

```js
// ...

renderTaskInlineError(task) {
  if (task.errors.title) {
    return (
      <div className="inline-error alert alert-danger">
        {task.errors.title.join(', ')}
      </div>
    );
  } else {
    return null;
  }
}

// ...
```

And then the ``finalAddTask`` function will be like following:

```js
// ...

handleAddTask() {
  this
  .state
  .project
  .tasks_attributes
  .push(Object.assign({}, this.emptyTask));

  this.setState({ project: this.state.project });
}

// ...
```

This is also quite simple. We can simply add an empty task to the state project tasks_attributes and reset back the state, then leave React to handle the view. It's really elegant!

The last 2 functions that we will need to implement is the ``handleCancel``:

```js
// ...

handleCancel() {
  this.props.history.push('/projects');
}

// ...
```
which is pretty straight forward, we can simply redirect the user back to the projects listing if the cancel button got hit.

And the ``handleFormSubmit`` function to handle for both cases: create a new project and update an existing project:


```js
// ...

handleFormSubmit() {
  let submitMethod = this.state.project.id ? 'patch' : 'post';
  let url = this.state.project.id
    ? `/projects/${this.state.project.id}.json`
    : '/projects.json';

  axiosClient
    [submitMethod](url, {
      project: this.state.project
    })
    .then(response => {
      this.props.history.push('/projects');
    })
    .catch(error => {
      this.setState({ project: error.response.data });
    });
}

// ...
```

Let's dive into this function. First we assign a submit method depend on whether there is a project id. It will be a ``POST`` if there is no project id, otherwise, it will be a ``PATCH``, following RESTful standard defined by Rails.

Next is the URL, for ``create`` and ``update``, the URL will be different. For ``create``, it will be simply ``/projects.json``, whereas for update it will be ``/porjects/<id>.json``. 

We then use ``axiosClient`` to call the coresponding method (post/patch) with the submit params is the ``this.state.project``. Rails is very smart to handle this so that it can save the entire project and all its tasks in 1 shot. Note that throughout the article, we always use ``tasks_attributes`` instead ``tasks``. Again, this is following Rails standard to save a nested model into database.

With the submission, there would be 2 cases, success and failure. 

For success case, we can simply redirect the user back to the project listing. You could also redirect user to the project view page. However, in this scope of this article, we will keep it simple. 

For the failure case (status code ``422`` - Rails ROLLBACK) we would extract out the response data returned by Rails and reassign it back to the main state so that React can handle the view rendering. This response back from Rails carries the ``errors`` attribute in the ``project`` and each ``task`` hence it will be display approriately in the view itself.

To make the inline error looks nice, we could also add an ``src/ProjectForm/Index.css`` with the content below:

```css
div.ProjectForm div.inline-error {
  padding: 5px;
  border-radius: 0 0 4px 4px;
  z-index: -1;
  top: -2px;
  position: relative;
}
```

and import it at the head of ``ProjectForm`` component:

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
  <img src='/public/gifs/reactjs-nested-form.gif' alt="ReactJS Nested Form" style='display:inline;'/>
</p>

The source file for this article is made publicly available on the following URLs:

[https://github.com/jameshuynh/blog-codes/tree/master/reactjs-nested-form](https://github.com/jameshuynh/blog-codes/tree/master/reactjs-nested-form)
