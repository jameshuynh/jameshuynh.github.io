---
layout: post
title: Nested Model Form
date: 2016-11-21
comments: true
categories: [rails, form]
tags: [rails, form]
excerpt_separator: <!-- more -->
---

Rails's abilty to build form quickly is great. However, ability to create a nested model form is not intuitive. This kind of task looks hard at first but if you know how to do it, it is pretty simple. In this article, I will quickly go through how we can build nested model form in Rails just by using jQuery.

<!-- more -->

## Setup Rails project and models

We start by setting up a new rails project

```bash
rails new project-management --database=mysql
```

Then change directory to the new project folder and create database:

```bash
cd project-management
bundle exec rake db:create
```

Let add models to represent a project management tool. This includes ``Project`` model and ``Task`` model.

```bash
rails g model Project name description:text
rails g model Task name is_done:boolean
```

Next, add ``has_many`` relationship in ``Project`` model:

```ruby
# project.rb
class Project < ApplicationRecord
  has_many :tasks, dependent: :destroy
end
```

## Build a Project and Tasks form

First, create a *scaffold* controller for ``Project`` model:

```bash
rails g scaffold_controller projects name description:text
```

After running the command, we will have a partial view ``_form.html.erb``, which we would need to edit. Our task is making this form display a sub-form for project's tasks.

Let's add the following standard code to ``_form.html.erb`` in order to display a sub-form to create project tasks:

```bash
<!-- _form.html.erb -->
...

<div class='tasks'>
  <div class='tasks-sub-form'>
    <%= f.fields_for :tasks do |task_form| %>
      <div class='field'>
        <%= task_form :name %>
        <%= task_form :name %>
      </div>

      <div class='field'>
        <%= task_form :is_done %>
        <%= task_form :is_done %>
      </div>
    <% end %>
  </div>
</div>
```

If you run this form now, you will notice that the input's name of ``name`` attribute and ``is_done`` attribute are: ``project[tasks][name]`` and ``project[tasks][is_done]``. It looks alright, however, Rails would not be able to understand this type naming when submit to the controller. We would need to add ``accepts_nested_attributes_for`` to the tasks relationship so that it would render the tasks form fields name correctly.

```ruby
# project.rb
class Project < ApplicationRecord
  has_many :tasks, dependent: :destroy
  accepts_nested_attributes_for :tasks
end
```

The attribute name would become: ``project[tasks_attributes][0][name]`` and ``project[tasks_attributes][0][is_done]``

Up to this point, the nested form is pretty much complete, but you still need to have 2 more things to the form complete:

- Add a task Link
- Remove a task Link

## Add Task Button

Let go ahead and add an ``Add a task`` link:

First, let's add a rails helper with the code below:

```ruby
module ApplicationHelper
  # f is a form object
  def link_to_add_fields(name, f, association, opts={})
    # creaate a new object given the form object, and the association name
    new_object = f.object.class.reflect_on_association(association).klass.new

    # call the fields_for function and render the fields_for to a string
    # child index is set to "new_#{association}, which would then later
    # be replaced in in javascript function add_fields
    fields = f.fields_for(association,
        new_object,
        :child_index => "new_#{association}") do |builder|
      # render partial: _task_fields.html.erb
      render(association.to_s.singularize + "_fields", f: builder)
    end

    # call link_to_function to transform to a HTML link
    # clicking this link will then trigger add_fields javascript function
    link_to_function(name,
      h("add_fields(this,
        \"#{association}\", \"#{escape_javascript(fields)}\");return false;"),
      class: 'btn btn-success')
  end

  def link_to_function(name, js, opts={})
    link_to name, '#', opts.merge({onclick: js})
  end
end
```

Next, add a javascript function ``add_fields`` in application.js. This javascript function would take the sub fields content (``content``) and append it into the desired container.

```js
function add_fields(link, association, content) {
  var new_id = new Date().getTime();

  // find the new_ + "association" that was defined in Rails helper
  var regexp = new RegExp("new_" + association, "g");

  // find the container and append in the sub field content
  $(link).prev().append(content.replace(regexp, new_id));
  return false;
}
```

Next, make use of ``link_to_add_fields`` function to make a link to add project tasks:

```erb
<!-- _form.html.erb -->
...

<div class='tasks'>
  ...
  <%= link_to_add_fields("Add Task", f, :tasks) %>
</div>
```

Lastly, you would need to move the code in ``fields_for`` to file ``_tasks_field.html.erb`` so that it can be shared between ``_form.html.erb`` and the ``link_to_add_fields`` function to render new field:

```erb
<!-- _tasks_field.html.erb -->

<div class='fields'>
  <h3>Task</h3>
  <div class='field'>
    <%= f.label :name %>
    <%= f.text_field :name %>
  </div>

  <div class='field'>
    <%= f.label :is_done %>
    <%= f.check_box :is_done %>
  </div>
</div>
```

Then, the code in ``_form.html.erb`` would be shorten to below:

```erb
<!-- _form.html.erb -->
...

<div class='tasks'>
  <div class='tasks-sub-form'>
    <%= f.fields_for :tasks do |task_form| %>
      <%= render "task_fields", f: task_form %>
    <% end %>
  </div>
  <%= link_to_add_fields("Add Task", f, :tasks) %>
</div>
```

If we try to submit the form now, we would notice that Rails is rejecting the attribute ``tasks_attributes``. To let it accept ``tasks_attributes``, we would need to add ``task_attributes`` and all its sub fields to ``project_params`` strong parameter function. We would also need to have ``id`` and ``_destroy`` so that the associated tasks can get updated or removed later on.

```ruby
# projects_controller.rb

def project_params
  params.require(:project).permit(
    :name,
    :description,
    tasks_attributes: [
      :name, :is_done, :id, :_destroy
    ])
end
```

## Remove Task Button

We would need to display a remove button next or below to each of the task group fo fields. Once clicking on this button, it would set hidden ``_destroy`` field to be ``1`` and at the same time hide this task group from the the collection. ``_destroy`` is a special field in rails association that mark an association instance as removed and would be deleted once the parent object get saved.

```erb
<!-- _task_fields.html.erb -->
...

<%= f.hidden_field :_destroy %>
<%= link_to "[Remove Task]", '#',
    onClick: "removeField(this); return false;" %>
```

And create a javascript function called ``removeField`` to change ``_destroy`` hidden value to be 1 and fade out the field:

```js
function removeField(link) {
  $(link).prev("input[type=hidden]").val("1");
  $(link).closest(".fields").fadeOut();
}
```

Then we would need to change our ``accept_nested_fields_for`` that we added in earlier to have ``_destroy`` field recognised. Rails will then be able to remove the task correctly if ``_destroy`` is marked as ``1``.

```ruby
# project.rb

accepts_nested_attributes_for :tasks, allow_destroy: true
```

## Conclusion

This is done for now. You now have a beautiful nested model form between project and tasks. 

Here is the final demo

<p style='text-align:center;' markdown='1'><img src='/public/gifs/nested_model_form.gif' alt="Final Demo" style='display:inline;'/></p>

There are still room for further extension though:

- Validate name of added task
- Enforce a minium number of tasks
- Enforce a maxium number of tasks
- Prepopulate N tasks before hand when you first load the form.

As usual, the source file for this article is made publicly available on the following URLs:

[https://github.com/jameshuynh/blog-codes/tree/master/project-management](https://github.com/jameshuynh/blog-codes/tree/master/project-management)
