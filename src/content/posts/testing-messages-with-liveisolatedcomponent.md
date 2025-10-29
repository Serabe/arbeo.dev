---
title: "Testing messages with LiveIsolatedComponent"
date: 2025-01-02
categories: 
  - "elixir"
  - "phoenix"
  - "liveview"
tags: 
  - "elixir"
  - "testing"
  - "components"
image: "[[attachments/EB4D6AC2-270F-4845-8EA0-695CFEBB8823.png]]"
imageOG: false
hideCoverImage: false
hideTOC: false
targetKeyword:
draft: false
---

## Two types of messages

There are two ways a component can send a message to their parent live view. First, any event dispatched from the browser (`phx-click`, `phx-submit`, etc.) will be handled in [LiveView.hanlde\_event/3](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html#c:handle_event/3) in their parent live view unless a `phx-target` specifies otherwise. For example, the following functional component sends a `"clicked"` event to their live view.

```
def my_button(assigns) do
  ~H"""
  <button type="button" phx-click="clicked">
    <%= render_slot(@inner_block) %>
  </button>
  """
end
```

In this component:

- `phx-click="clicked"` tells LiveView to send the `"clicked"` event when the button is pressed.

- The absence of a `phx-target` attribute makes the event be handled by the parent live view.

- The parent live view can handle this message with the callback `handle_event("clicked", params, socket)`.

There is a second option, but this one requires a stateful LiveView component, and is just using `send(self(), message)`. Messages sent this way are managed by [`LiveView.handle_info/2`](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html#c:handle_info/2). The following live component opens and closes a panel, and let the parent view know about it.

```
defmodule MyAppWeb.MyTogglingPanel do
  use MyAppWeb, :live_component

  def mount(socket), do: {:ok, assign(socket, :open, false)}

  def render(assigns) do
    ~H"""
    <div id={@id}>
      <button phx-click="toggle" phx-target={@myself}>
        <%= render_slot(@button, %{open: open}) %>
      </button>
      <div :if={@open}>
        <%= render_slot(@panel) %>
      </div>
    </div>
    """
  end

  def handle_event("toggle", _params, socket) do
    socket = update(socket, :open, & not &1)

    send(self(), {:panel_toggled, Map.take(socket.assigns, [:id, :open])})

    {:noreply, socket}
  end
end
```

This component is a bit more complex. Let's dive in:

- `mount/1` sets the default value for `:open` to `false`.

- `render/1` displays a button that sends a `"toggle"` event to the component itself, thanks to `phx-target={@myself}`. Also, conditionally renders a panel based on the value of `@open`.

- `handle_event/3` toggles the value of `:open` and `send/2` a message to the parent live view.

- The parent live view can handle this message with `handle_info({:panel_toggled, %{id: id, open: open})`.

**Note:**  Messages can’t be sent directly between components. A workaround involves using `send_update/2`. For more flexibility and standardized messaging, check out the [live\_view\_events](https://dockyard.com/blog/) library.

## Handling events and calls

**Focus testing on behaviour, not implementation**. This means that testing which messages are being sent out by the component, rather than the events being received by the component itself. If you think about it, it makes sense. We don't care about the name of the event the toggles the panel, we just care that when pressing the button, the panel does get opened / closed, and that the right message is being sent to the parent live view.

LiveIsolatedComponent provides five key assertions for testing messages sent to the parent live view:

- [`assert_handle_event4`](https://hexdocs.pm/live_isolated_component/LiveIsolatedComponent.html#assert_handle_event/4) and [`refute_handle_event/4`](https://hexdocs.pm/live_isolated_component/LiveIsolatedComponent.html#refute_handle_event/4).

- [`assert_handle_info/3`](https://hexdocs.pm/live_isolated_component/LiveIsolatedComponent.html#assert_handle_info/3) and [`refute_handle_info/3`](https://hexdocs.pm/live_isolated_component/LiveIsolatedComponent.html#refute_handle_info/3).

- [`assert_handle_event_return/2`](https://hexdocs.pm/live_isolated_component/LiveIsolatedComponent.html#assert_handle_event_return/2), for which I haven’t found any interesting case, given the return is from a mock view we use only for testing. That said, it was easier to add it than to wait for someone to ask for it ;P

For both handle\_event and handle\_info, you can assert on whether a message was sent, its name, and even its parameters. Check the documentation for a full list, as we'll explore only a few examples below.

## Testing events

```
test “sends handle event when clicked” do
  {:ok, view, _html} = live_isolated_component(&my_button/1, slots:
    [
      inner_block: slot() do ~H[Click me!] end
    ])

  view |> element(“button”) |> render_click()

  # assertion here
end
```

Possible assertions:

- `assert_handle_event(view)` verifies that an event has been received, regardless of its name or parameters.

- `assert_handle_event(view, “clicked”)` also checks for the event name.

- `refute_handle_event(view)` ensures no event was received.

- `refute_handle_event(view, “toggle”)` ensures no `"toggle"` event was received. As `my_button/1` sends a `"clicked"` event, this assertion passes.

## Testing messages sent with `send`

```
test “a message is sent to view each time the panel is toggled” do
  {:ok, view, _html} = live_isolated_component(MyAppWeb.MyTogglingPanel, slots:
    [
      button: slot(let: %{open: open}) do
        if open, do: ~H[Close], else: ~H[Open]
      end,
      pannel: slot() do ~H[Some interesting content] end
    ])

  view |> element(“button”) |> render_click()

  # assertion here
end
```

In this case, the assertions can be:

- `assert_handle_info(view)` just checks that any message has been received.

- `assert_handle_info(view, {:panel_toggled, _params})` matches a specific event name.

- `assert_handle_info(view, {:panel_toggled, params})` let us grab the value of the parameters received for further assertions.

- `assert_handle_info(view, {:panel_toggled, %{open: true}})` validates the parameters themselves too.

- `refute_handle_info(view)` ensures no message has been received.

- `refute_handle_info(view, {:panel_toggled, _params})` verifies that no `:panel_toggled` message has been received.

- Checking that no message with some exact param has been received is more rare, but can be useful when some kind of id is in there and you expect messages from some ids but no others.

## Conclusion

We've discovered how powerful message assertions in LiveIsolatedComponent can be. After these few posts, we are ready to test 99,99% of our components. There are a few more cards in LiveIsolatedComponent's sleeves, but that is a topic for some other posts coming this year. Stay tuned for patterns and advanced options to level up your LiveView testing skills!
