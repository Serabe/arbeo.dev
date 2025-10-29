---
title: "Better Tests, Zero Drama: Smarter LiveIsolatedComponent Patterns"
date: 2025-01-08
description: A list of useful patterns to use with LiveIsolatedComponent
categories: 
  - "elixir"
  - "phoenix"
  - "liveview"
tags: 
  - "elixir"
  - "testing"
  - "components"
image: "[[attachments/DALL·E-2025-01-03-02.12.08-An-.webp]]"
imageOG: false
hideCoverImage: false
hideTOC: false
targetKeyword:
draft: false
---

## Introduction

Using `LiveIsolatedComponent` to test your components in isolation simplifies the process by removing the need to either write a mock view or set up an existing one. Now that testing is more streamlined, we can focus on improving the testing experience with additional goals related to readability and maintainability.

The following component displays a list of posts in a table, and it's the one we'll test in this post.

```
defmodule MyApp.Components.PostsTable do
  use MyApp, :live_component

  def render(assigns) do
    ~H"""
    <table>
      <thead>
        <tr>
          <td></td>
          <td>Title</td>
          <td>Author</td>
          <td>Published</td>
        </tr>
      </thead>
      <tbody>
        <.post_item
          :for={post <- @posts}
          post={post}
          current_user={@current_user}
          timezone={@timezone}
        />
      </tbody>
    </table>
    """
  end

  def post_item(assigns) do
    ~H"""
    <tr data-test-post={@post.id}>
      <td data-test-special>
        <%= if @post.author == @current_user do %>
          <span class="mine">Mine</span>
        <% end %>
      </td>
      <td data-test-title>
        <%= @post.title %>
      </td>
      <td data-test-author>
        <%= author(@post) %>
      </td>
      <td data-test-published>
        <%= shift_to_timezone(@post.published_at, @timezone) %>
      </td>
    </tr>
    """
  end
end
```

The component receives a list of `posts`, a `current_user`, and a `timezone`. For the sake of this article, all parameters are required. The component renders a table displaying the post title, author, and published date shifted to the specified timezone. It also highlights posts written by the current user.

## data-test selectors

Notice the use of the `data-test` and `data-test-x` attributes in the component. These are called data-test attributes (sometimes, naming isn’t hard) and are meant to decouple your tests from CSS classes or complex IDs. If the HTML engine were easily pluggable, we could create plugins to remove these attributes in production builds. Let’s add that to our wishlist for this year. 🤞

There are two types of data-test selectors:

- **With values**: Used for filtering selections (e.g., `[data-test-post=some-id]`) or checking internal states. However, relying on internal state for tests can lead to brittle tests, so use this sparingly.

- **Without values**: Used for accessing unique or non-repeated elements (e.g., `[data-test-author]`). These can often be combined with value-based selectors for specificity (e.g., `[data-test-post=some-id] [data-test-author]`).

Data-test attributes can (and should) be nested. Avoid turning them into something like [BEM](https://getbem.com/); just keep them simple and structured hierarchically.

## Default Parameters for the Component.

`LiveIsolatedComponent` is probably the piece of code written outside my workplace that I've used the most in work-related projects. Using it daily has helped me form strong opinions on how to write better tests with it.

A key goal when writing tests is readability. Tests should avoid noise and focus only on the data needed to replicate the behavior being tested. For example, if we’re testing that the correct author is displayed, details like the `current_user` or `timezone` are irrelevant.

To simplify tests, I use a wrapper function to provide default values for the component’s required attributes:

```
def live_posts_table(opts \\ []) do
  opts =
    opts
    |> Map.put_new(:assigns, %{})
    |> Map.update!(:assigns, fn value ->
      assigns
      |> Map.put_new(:posts, [])
      |> Map.put_new(:timezone, "EST")
      |> Map.put_new_lazy(:current_user, fn -> insert(:user) end)
    end)

  live_isolated_component(MyApp.Components.PostsTable, opts)
end
```

I’ve written about the `put_new` + `update!` pattern on [DockYard’s blog](https://dockyard.com/blog/2024/04/02/a-better-way-to-update-nested-maps-and-keywords-in-elixir). In short, it improves developer experience (DX) when updating map values. With this pattern, tests can focus solely on passing relevant data, without worrying about boilerplate.

Here’s an example comparing the usual approach and the helper function:

**Without Helper:**

```
test "post title gets render (usual way)" do
  title = "Nice Hotels near Madrid"
  post = insert(:post, title: title)
  current_user = insert(:user)
  timezone = "CEST"

  {:ok, view, _html} = live_isolated_component(
    MyApp.Components.PostsTable,
    assigns: %{
      current_user: current_user,
      posts: [post],
      timezone: timezone
    })

  assert has_element?(view, "[data-test-post=#{post.id}] [data-test-title]", title)
end
```

**With Helper:**

```
test "post title gets render (usual way)" do
  title = "Nice Hotels near Madrid"
  post = insert(:post, title: title)

  {:ok, view, _html} = live_posts_table(
    assigns: %{posts: [post]}
  )

  assert has_element?(view, "[data-test-post=#{post.id}] [data-test-title]", title)
end
```

Key improvements:

- Unnecessary details like `current_user` and `timezone` are omitted.

- The `live_isolated_component` call is simplified, removing module name repetition throughout the test module.

- It reduces noise, ensuring that the only relevant part is emphasized—the specific `post` and its `title`.

## Extracting Accessors and Interactions

To further improve tests, abstract away selectors and interactions into functions. I recommend using private functions within the test module. If these functions are needed in multiple modules, extract them into a shared helper module. That said, I find this case to be useful in pretty rare occasions, though this is a discussion for another post.

Here’s an example for selectors:

```
defp post_selector(post) do
  "[data-test-post=#{post.id}]"
end

defp title_selector(post) do
  post_selector(post) <> " [data-test-title]"
end

defp has_title?(view, post, expected_title) do
  has_element?(view, title_selector(post), title)
end
```

First, selectors are always functions. One might think that if it is a static selector (like `[data-test-title]`), we might prefer module attributes. Always going with functions let's me reduce the thinking, and keep consistent. I'm that lazy 😜

**Note:** I don't exactly write `has_title?`\-like helpers. I'll discuss this at a later post.

### Interactions

For interactions, I always return the `view` so I can easily pipe interactions together, not only in tests, but in other helpers too. For example, if we were in a `details`\-like component, we could have the following helper function:

```
def toggle(view) do
  view
  |> element("[data-test-toggler]")
  |> render_click()

  view
end
```

This approach allows chaining interactions seamlessly, improving both test clarity and maintainability.

## Conclusion

While `LiveIsolatedComponent` already provides significant improvements for testing, these additional techniques can make your tests even more expressive and maintainable. Combining these practices will result in cleaner tests and a smoother development experience.
