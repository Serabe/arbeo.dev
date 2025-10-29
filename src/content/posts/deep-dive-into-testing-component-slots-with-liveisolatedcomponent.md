---
title: "Deep Dive into Testing Component Slots with LiveIsolatedComponent"
date: 2024-11-19
categories: 
  - "elixir"
  - "phoenix"
  - "liveview"
image: "[[attachments/DALL·E-2024-11-18-21.26.37-An-.webp]]"
imageOG: false
hideCoverImage: false
hideTOC: false
targetKeyword:
draft: false
---

## Introduction

In this guide, we'll explore how to effectively test component slots using the `LiveIsolatedComponent` library. By understanding how slots work, you'll be able to make your Phoenix LiveView components more flexible and customizable while maintaining testability.

## What Are Slots?

Slots are a powerful way to customize parts of a component's template. Think of attributes as basic values you work with in programming (like numbers, strings, and functions), while slots are like special placeholders that let you add custom content to a component.

You can set attributes for these slots and pass values to them. If you’re new to slots, check out [the official documentation on slots](https://hexdocs.pm/phoenix_live_view/Phoenix.Component.html#module-slots), and come back later to dive into my upcoming series on the topic.

## Testing Simple Slots

Let's start by testing a simple functional component with a slot:

````
@doc """
```heex
<.pill>Click me</.pill>
```
"""
def pill(assigns) do
  ~H"""
  <button class="pill">
    <%= render_slot(@inner_block) %>
  </button>
  """
end
````

This component creates a button with the `pill` class and displays whatever content you pass to it. Now, let’s write a test for it:

```
test "renders default slot" do
  {:ok, view, _html} = live_isolated_component(&MyAppWeb.Functional.pill/1, slots: [
    inner_block: slot do
      ~H[Hello there]
    end
  ])
  
  assert has_element?(view, "button", "Hello there")
end

# If the `live_isolated_component` above was a template,
# it would look like this.
def template(assigns) do
  ~H"""
  <.pill>Hello there</.pill>
  """
end
```

Here’s what’s happening:

- We are testing a functional component, so we pass the function as the first argument.

- We only need to pass the `:slots` option, as there are no `:assigns` being used in the component.

- `:slots` is a keyword list where keys are slot names and values are slot blocks.

- We use the `:inner_block` name because that’s what Phoenix LiveView uses.

- For the value, we use the [`LiveIsolatedComponent.slot/2`](https://hexdocs.pm/live_isolated_component/LiveIsolatedComponent.html#slot/2) macro, which sets up a slot block with a HEEX template.

## Testing Attributes in Slots

You can easily pass attributes to the `slot/2` macro as the first argument. Just provide a keyword list with the attributes:

```
slot(arg_1: "first_value", arg_2: true, arg_3: 3) do
  ~H"""
  <div>Some content here</div>
  """
end
```

We’ll dive into an example of this in the next section, where we’ll discuss `:let`.

## Testing `:let`

You can pass values into slots, which are _sent_ as the second argument to `render_slot/2` and _received_ via the `:let` attribute of the slot. In LiveIsolatedComponent, this attribute is simply called `let`. Here’s a quick example:

```
def list(assigns) do
  ~H"""
  <ul>
    <%= for i <- @collection do %>
      <li id={i.id} class={@item |> hd() |> Map.get(:class)}>
        <%= render_slot(@item, %{item: i}) %>
      </li>
    <% end %>
  </ul>
  """
end
```

This component renders a list when used like in the following snippet:

```
def render(assigns) do
  ~H"""
  <.list collection={~w(hello darkness my old friend)}>
    <:item :let={%{item: item}} class="list-decimal">
      <%= item %>
    </:item>
  </.list>
  """
end
```

To test this, we use the `slot` macro again:

```
test "list component" do
  {:ok, view, _html} = live_isolated_component(
    &list/1,
    assigns: %{collection: ~w(hello darkness my old friend)},
    slots: [
      item: slot(let: item, class: "list-decimal") do
        ~H[<%= item %>]
      end
    ]
  )
  
  Enum.each(~w(hello darkness my old friend), fn item ->
    assert has_element?(view, ".list-decimal", item), "#{item} is rendered"
  end)
end
```

The `let` attribute is special—whatever value gets passed in gets bound to the variable name given in `let`, and you can use it within the slot’s template. You can even use destructuring if needed!

In this test:

- `let: item` binds the value passed into the slot by `render_slot/3` to the `item` variable. This allows us to use `item` directly within the slot’s template, enabling dynamic rendering based on the slot’s context.

- The `class: "list-decimal"` attribute is passed to the slot, setting the `class` for the `<li>` elements surrounding the slot's content. This attribute is used in the component's template to apply CSS styling.

- The `assert has_element?` checks verify that each item from the collection is rendered within elements that have the `"list-decimal"` class, ensuring correct rendering and styling.

## Passing Multiple Slots with the Same Name

One cool feature of Phoenix LiveView slots, often unknown or unused, is that you can pass multiple slots with the same name. This is handy, for example, in a table component:

```
def table(assigns) do
  assigns = assign_new(assigns, :cell, fn -> [] end)
  
  ~H"""
  <table>
    <tbody>
    <%= for item <- @items do %>
      <tr>
        <%= for cell <- @cell do %>
          <td><%= render_slot(cell, item) %></td>
        <% end %>
      </tr>
    <% end %>
    </tbody>
  </table>
  """
end
```

In this example, you can pass several `cell` slots, which get rendered **individually**. We’ll cover a more complex example in a future post.

Here’s how you’d use the `table` component:

```
def render(assigns) do
  ~H"""
  <.table items={@people}>
    <:cell :let={person}><%= person.name %></:cell>
    <:cell :let={person}><%= person.age %></:cell>
  </.table>
  """
end
```

LiveIsolatedComponent supports passing multiple slots in two ways: as an array of slots or by specifying multiple slots with the same name:

```
test "renders as many cells as passed in" do
  {:ok, view, _html} = live_isolated_component(
    &table/1,
    assigns: %{items: [insert(:person, name: "Dai Vernon", age: 98)]},
    slots: [
      cell: slot(let: person) do
        ~H[<%= person.name %>]
      end,
      cell: slot(let: person) do
        ~H[<%= person.age %>]
      end
    ]
  )
  
  assert has_element?(view, "td", "Dai Vernon")
  assert has_element?(view, "td", "98")
end
```

I personally prefer the latter, as it is closer to how it is done in HEEX templates.

## Conclusion

And that’s a wrap on our deep dive into testing slots! We’ve covered how to test simple slots, pass attributes to them, use `:let` for binding values, and even handle multiple slots with the same name. With these techniques, you can make your LiveView components even more flexible while still being easily testable.

Understanding and mastering slots opens up a world of possibilities for component customization. As you continue to build and refine your LiveView components, these insights will help ensure your components behave exactly as expected and handle complex scenarios with ease. Soon I'll start another series on slots, their basic usage and some cool techniques to unleash all the power slots can give you.

Next up, we’ll tackle how to test if your components are sending the right events or messages to their parent LiveView. Stay tuned for more tips and tricks to level up your LiveView testing game!
