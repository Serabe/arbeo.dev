---
title: "Effortless Component Testing your first component in isolation with LiveIsolatedComponent"
date: 2024-10-28
categories: 
  - "elixir"
  - "phoenix"
  - "liveview"
tags: 
  - "elixir"
  - "testing"
  - "components"
image: "[[attachments/DALL·E-2024-10-27-02.15.56-Fea-e1729992190382.webp]]"
imageOG: false
hideCoverImage: false
hideTOC: false
targetKeyword:
draft: false
---

## Why Use LiveIsolatedComponent?

Throughout my career, I've worked with various technologies. I began as a back-end developer, transitioned to front-end technologies—especially Ember.js—and now I'm back to server-side development, focusing on Phoenix LiveView.

While I enjoy using LiveView, I missed some features from my front-end experience. One key feature I missed was the ability to test components in isolation.

In the past, testing components meant either testing within an existing view or creating a new `MockView` for the test. The first approach required setting up everything the view needed, even if it wasn't relevant to the component being tested. The second approach often led to messy tests because it was challenging to pass all the necessary data, leading to multiple rendering paths like this:

```
defmodule MockView do
  use MyAppWeb, :live_view
  
  def mount(params, %{render_version: version}=session, socket) do
    {:ok, assign(socket, :render_version, version)}
  end
  
  def render(%{render_version: "a"}=assigns) do
    ~H"""
    <.my_component attr="one value" />
    """
  end
  
  def render(%{render_version: "b"}=assigns) do
    ~H"""
    <.my_component attr="another value" />
    """
  end
end
```

In this setup, I would pass some kind of ID or render path identifier in the session to match in `render/1`. While this worked, it had several issues:

- As the test suite grew, it became harder to understand because the view and the test were separated from each other.

- Some values, especially non-serializable ones, couldn’t be passed through the session. This was a problem for components with multiple slots, needing different renders for each combination.

- Re-implementing tests to verify that the correct events or messages were sent by the component was repetitive and inefficient.

To address these issues, I developed [live\_isolated\_component](https://github.com/Serabe/live_isolated_component/), which solves all these problems. Let’s dive into the first example.

## Introducing the Component

In this post, I'll guide you through the basics of testing components with `live_isolated_component`, focusing on rendering and updating a component. Here’s our example component:

```
defmodule MyAppWeb.SimpleComponent do
  use MyAppWeb, :live_component
  @moduledoc """
  Simple module that receives a value and an updater function
  and updates the value using the given updater each time the
  user clicks a button.
  
  ## Example
  
  ```heex
  <.live_component
    module={MyAppWeb.SimpleComponent}
    id="an-id"
    value={5}
    updater={fn x -> x * 2 end}
    />
  ```
  """
  
  def render(assigns) do
    ~H"""
    <div data-test="simple-component">
      <span data-test="value"><%= @value %></span>
      <button phx-click="update_value" phx-target={@myself}>
        Update
      </button>
    </div>
    """
  end
  
  def handle_event("update_value", _params, socket) do
    {:noreply, update(socket, :value, socket.assigns.updater)}
  end
end
```

## Testing the Component

Let’s start with a simple test to verify that the passed-in value is displayed correctly:

```
defmodule MyAppWeb.SimpleComponentTest do
  use MyAppWeb.ConnCase
  
  import LiveIsolatedComponent
  
  test "value is displayed" do
    {:ok, view, _html} =
      live_isolated_component(
        MyAppWeb.SimpleComponent,
        assigns: %{value: 5, updater: fn x -> x + x end}
      )
      
    assert has_element?(view, "[data-test=value]", "5")
  end
end
```

Here’s how it works:

- Import the `LiveIsolatedComponent` module to gain access to its functions and macros, including `live_isolated_component`.

- `live_isolated_component` takes two parameters:
    1. The first argument can be a module (for stateful live components) or a function (for functional components).
    
    3. The second argument is a list of options. In this example, we use the `:assigns` option to pass values as attributes to the component. We will dive into other options in other posts.

- The `live_isolated_component` macro returns values similar to `live_isolated` or `live` macros and can be used in the same way. In the example, we use `has_element?/3` to verify that the value is displayed correctly.

## Interacting with the Component

Next, let’s test the behavior of updating the value when the user clicks the button:

```
test "value gets updated when button is clicked" do
  {:ok, view, _html} =
    live_isolated_component(
      MyAppWeb.SimpleComponent,
      assigns: %{value: 5, updater: fn x -> x + x end}
    )
      
  assert has_element?(view, "[data-test=value]", "5")
  
  view |> element("button") |> render_click()
  
  assert has_element?(view, "[data-test=value]", "10")
end
```

In this test:

- We use LiveView testing helpers `element` and `render_click` to simulate clicking the button.

- After clicking the button, we check that the value is updated correctly in the DOM.

## Handling External Changes

Finally, let’s test how the component reacts to changes in the passed-in attributes:

```
test "component forgets about the updated value if the attribute changes" do
  {:ok, view, _html} =
    live_isolated_component(
      MyAppWeb.SimpleComponent,
      assigns: %{value: 5, updater: fn x -> x + x end}
    )
  
  view |> element("button") |> render_click()
  assert has_element?(view, "[data-test=value]", "10")
  
  live_assign(view, :value, 2)
  
  assert has_element?(view, "[data-test=value]", "2")
end
```

In this test:

- We first simulate clicking the button to update the value.

- Then, we use `live_assign` to change the `:value` attribute to `2`.

- Finally, we verify that the component reflects this updated value in the DOM.

## Next in this Series

We've only scratched the surface of `LiveIsolatedComponent`. Since I started using it at work, the number of tests and coverage has increased significantly due to its lower barrier to entry. In upcoming posts, we’ll explore how to handle slots, verify that the component sends the correct events and calls, and customize parts of the view where the component is rendered.

## A Final Challenge

In the last example, we changed the `:value` to `2`. This choice wasn’t random. Can you figure out what would happen if we changed it to `1`? Stay tuned for the solution!
