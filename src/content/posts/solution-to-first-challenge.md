---
title: "Solution to First Challenge"
date: 2024-10-31
categories: 
  - "elixir"
  - "phoenix"
  - "liveview"
tags: 
  - "elixir"
  - "testing"
image: "[[attachments/E865E7BF-DF33-477E-AB7C-377AF74E9248.png]]"
imageOG: false
hideCoverImage: false
hideTOC: false
targetKeyword:
draft: false
---

## Understanding the Test

The challenge was to explain why, in the following test, the value is set to `2` instead of `1`:

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

The reason the value is set to `2` in `live_assign` instead of `1` is to show how the component behaves when the value changes externally. In this test, we're checking that the component correctly updates the value it displays when a new value is assigned. It’s also important to note that the element matching the selector `[data-test=value]` only displays the content of `@value`.

## How `has_element?/3` Works

The `Phoenix.LiveViewTest.has_element?/3` function ([see code](https://github.com/phoenixframework/phoenix_live_view/blob/3d46bdf6916119727897463413c6f11c06b15d63/lib/phoenix_live_view/test/live_view_test.ex#L1035-L1037)) creates an element with a `text_filter` set to either `"10"` or `"2"` in our example:

```
def has_element?(%View{} = view, selector, text_filter \\ nil) do
  has_element?(element(view, selector, text_filter))
end
```

This function calls `Phoenix.LiveViewTest.has_element?/1` ([see code](https://github.com/phoenixframework/phoenix_live_view/blob/main/lib/phoenix_live_view/test/live_view_test.ex#L1050)):

```
def has_element?(%Element{} = element) do
  call(element, {:render_element, :has_element?, element})
end
```

The `:render_element` message triggers `:sync_render_element`, which then invokes `Phoenix.ClientProxy.select_node_by_text/4`. The key part here is:

```
filtered_nodes = Enum.filter(nodes, &(DOM.to_text(&1) =~ text_filter))
```

The use of `=~` ([the text-based match operator](https://hexdocs.pm/elixir/1.12.3/Kernel.html#=~/2)) can be problematic because it is not very precise. This makes tests unclear or gives misleading results.

## Explaining the Challenge

Let’s assume there’s a bug in our implementation where, once the value is set, updates to it are ignored. In our current test, `"10" =~ "10"` is true, so the test passes. However, when the value is updated `"10" =~ "2"` is false, which causes the test to fail.

Consider this modified test:

```
test "component forgets about the updated value if the attribute changes" do
  {:ok, view, _html} =
    live_isolated_component(
      MyAppWeb.SimpleComponent,
      assigns: %{value: 5, updater: fn x -> x + x end}
    )
  
  view |> element("button") |> render_click()
  assert has_element?(view, "[data-test=value]", "10")
  
  live_assign(view, :value, 1)
  
  assert has_element?(view, "[data-test=value]", "1")
end
```

In this version, the second assertion checks `"10" =~ "1"`, which falsely passes because of the previously mentioned bug. This means that if the text filter isn’t specific enough, the test might incorrectly pass or fail based on vague matches rather than exact values.

## Conclusion

This example shows why precision in testing is important. If `has_element?/3` relies on vague text matching, the tests can become unreliable and difficult to understand. Making sure that your tests check for exact values can help prevent misleading results and ensure that your components work as expected.

This also highlights the importance of having strong testing support and reliable testing practices. Poor testing practices can make development harder, lead to tests that are challenging to write and maintain, and ultimately lower the motivation to thoroughly test your application. This series aims to provide tools to improve the developer experience (DX) of testing, which will help keep your application well-tested.

In future posts, we will introduce new testing libraries that aim to improve the reliability of tests and make it easier to understand test failures. These improvements will help you find issues faster and understand why tests fail. Stay tuned for more insights on enhancing your testing strategy!
