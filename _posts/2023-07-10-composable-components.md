---
layout: post
title: Composable, Controllable and Chatty Phoenix Live-Components
comments: true
---

Writing live components in Elixir's Phoenix Framework is a breeze. It
allows us to hide internal state and mechanics with ease. However,
when it comes to writing helpful abstractions for components that
encapsulate functionality like parameter handling and still compose,
it can get messy. In this blog post, I will demonstrate how to
implement a controllable and chatty tabs component that can be
effortlessly composed with other components.


<!--more-->

To understand this blog post, the reader requires basic knowledge in
Elixir, the Phoenix Framework, as well as live components. The code
that is shown in the blog post is accessible in a GitHub repo, and the
story in this post is mirrored in the repo's history. Each
intermediate step is fully functional. Watch out for these banners:

>  [Open the project](https://github.com/smoes/reusable-elixir-components/) on GitHub.

## Requirements

In this blogpost we'll implement a tabs component. A tabs component is a
component that offers a navigation menu or bar with links. When a link
is clicked the content area of the component shows the relevant information.

The tabs component in this blog post has some non-trivial
requirements: The component...

1. should be composable with other components
2. encapsulates uri-parameter handling to persist the active tab in
   the uri
3. can be chatty, that is, it can inform the parent live
   view about state changes
4. is controllable from the parent live view


## Lets get basic at first

For the blogpost, I set up a fresh Phoenix project called `DemoWeb`
and created a live view called `LiveDemo`. This live view is served
via `/tabs`. In the render function of this live view I added dummy
code, to demonstrate how the component should be used later.

```elixir
defmodule DemoWeb.LiveDemo do
  use DemoWeb, :live_view

  alias DemoWeb.Components.Tabs

  def mount(_params, _session, socket) do
    {:ok, socket}
  end

  def render(assigns) do
    ~H"""
    <.live_component module={Tabs} id="tabs">
      <:tab label="Name" id={:name_tab}>Simon</:tab>
      <:tab label="Address" id={:address_tab}>742 Evergreen Terrace, Springfield</:tab>
    </.live_component>
    """
  end
end
```


I want the call of the tabs component to be as simple as possible. The
component can have tab slots, which consist of a label and an ID
attribute. The label will be used as the text for the tab link, while
the ID serves as an internal identifier for later use. The content is
enclosed within the inner block of the tabs.

The tabs component itself is implemented in the
DemoWeb.Components.Tabs module. I prefer to make the data structure
explicit whenever feasible and strictly use structs instead of
maps. The state of the tabs component is represented as follows:

```elixir
defmodule State do
  defstruct [:active_id, :tabs]

  alias __MODULE__

  def active_slot(%State{tabs: tabs, active_id: active_id}) do
    %Tab{slot: slot} = Enum.find(tabs, fn tab -> tab.id == active_id end)
    slot
  end

  def active?(%State{active_id: active_id}, tab) do
    active_id == tab.id
  end
end
```

The field `active_id` stores the ID of the currently active tab, while
tabs is the list of slots, represented by another struct:

```elixir
defmodule Tab do
  defstruct [:id, :label, :slot]
end
```

The function `active_slot` returns the slot for the active tab, given
a state, while `active?` indicates whether a given tab ID is currently
active.

The rendering of the component is kept simple:

```elixir
def render(assigns) do
  ~H"""
  <div class="flex flex-col space-y-5">
    <div class="flex flex-row space-x-3">
      <%= for tab <- @state.tabs do %>
        <div
          class={[
            "rounded-full p-3 pl-5 pr-5",
            if State.active?(@state, tab) do
              ["bg-white text-black"]
            else
              ["bg-black text-white"]
            end
          ]}
          phx-click="change_tab"
          phx-value-id={tab.id}
          phx-target={@myself}
        >
          <%= tab.label %>
        </div>
      <% end %>
    </div>
    <div>
      <%= render_slot(@state |> State.active_slot()) %>
    </div>
  </div>
  """
end
```

As we can observe, clicking on a tab link triggers a `change_tab` event, sending the tab's ID to the live component. An event handle function is then responsible for updating the active tab:

```elixir
 def handle_event("change_tab", %{"id" => raw_id}, socket) do
    id = String.to_existing_atom(raw_id)
    current_state = socket.assigns.state
    next_state = %State{current_state | active_id: id}
    {:noreply, socket |> assign(:state, next_state)}
  end
```

The attentive reader may have noticed that we skipped an important
part: how do we translate the slots and their attributes into the
defined structs? To accomplish this, we utilize the update method of
the live component. The `update` function is invoked after mounting a
live component and whenever the assigns of the live component's
instance change.

```elixir
def update(assigns, socket) do
  # check if the state was already initialized, c.f. GitHub
  unless initialized?(socket) do
    # call a helper method to translate into struct, c.f. GitHub
    tabs = make_tabs(assigns.tab)
    # make the first tab the active tab
    active = hd(tabs).id
    state = %State{active_id: active, tabs: tabs}
    {:ok, socket |> assign(:state, state)}
  else
    {:ok, socket}
  end
end
```

Our implementation of `update` takes assigns and the live component's
socket object (not that of the live view!) as arguments. Within the
assigns, we can find a list of slots in tab. Using this information,
we can update the assigns in the socket object of the live component.

> [Open the code](https://github.com/smoes/reusable-elixir-components/tree/03b849497be5aa3fb2cd485343525a6bef8b4438) of this basic variant on GitHub

Sadly this simple component doesn't fullfil any of the
given requirements.

## Passing State

It is evident that the basic version of the live component does not
fulfill requirements _2-4_. However, what about requirement _1_
(Composability)? Is the component composable? For instance, can we
place another tabs component within one of the tabs? Indeed, we can,
and it will function as expected. However, what we cannot do is pass
assigns from our parent live view and anticipate them to be updated in
the tabs. Let's examine this example:

```elixir
defmodule DemoWeb.LiveDemo do

  ...

  def render(assigns) do
    ~H"""
    <.live_component module={Tabs} id="tabs">
      <:tab label="Tab 1" id={:tab_1}><%= @counter %></:tab>
      <:tab label="Tab 2" id={:tab_2}>Tab 2</:tab>
    </.live_component>

    <p phx-click="increment">Increment</p>
    <p><%= @counter %></p>
    """
  end

  # event handler that increases the counter
  def handle_event("increment", _, socket) do
    next_socket = socket |> update(:counter, fn counter -> counter + 1 end)
    {:noreply, next_socket}
  end
end
```

In this example a counter gets updated when clicking on
`Increment`. Displaying the counter outside tabs component work
perfectly fine; it updates after click. Displaying the counter within
first slot of the tabs component however, doesn't. It will stay at the
initial value.

> [Open code for the non-working example](https://github.com/smoes/reusable-elixir-components/tree/a1cba53c11c61c16b42e73bb2036dc637e60694c) on GitHub

To fix that, we add a field `maybe_inner_state` to the `State` struct
and write an assign called state into our live component's state:

```elixir 
def update(assigns, socket) do
  maybe_state = Map.get(assigns, :state)

  unless initialized?(socket) do
    ...
    state = %State{active_id: active, tabs: tabs, maybe_inner_state: maybe_state}
    {:ok, socket |> assign(:state, state)}
  else
    state = socket.assigns.state
    updated_state = %State{state | maybe_inner_state: maybe_state}
    {:ok, socket |> assign(:state, updated_state)}
  end
end
```

The render slot call must be extended to pass the state to the slot.

```elixir
<%= render_slot(@state |> State.active_slot(), @state.maybe_inner_state) %>
```

Now that we have the state as a parameter we are handing to the slot,
Phoenix can keep track of changes. To make the counter example work,
we can now pass the assign like follows:

```elixir
<.live_component module={Tabs} id="tabs" state={@counter}>
  <:tab label="Tab 1" id={:tab_1} :let={counter}><%= counter %></:tab>
  <:tab label="Tab 2" id={:tab_2}>Tab 2</:tab>
</.live_component>
```

> [Open code for the working example](https://github.com/smoes/reusable-elixir-components/tree/ec6ca25227a2c8ff757c5241ff4a597bfece4348) on GitHub

## Introducing URI Prameters


As mentioned in the requirements section, we want our tabs component to utilize URI parameters to persist the active tab in the URI. Unfortunately, live components do not have a `handle_params` callback like live views do. How can we respond to URI changes without exposing implementation details in the live view's `handle_params`?

Fortunately, live components can implement hooks, and here's a surprise: we can register a hook for `handle_params`. This means we can define a function that is called after the live view's `handle_params` has been executed. The hook receives the same parameters as the original function: the parameters, the current URI, and the live view's socket. However, since this is not the socket of our live component, we need to find a way to modify the live component's state within the live view's socket assigns and propagate the changes.

To achieve this, we create a function called `init`, which is invoked in the `mount` method of the parent live view. This function initializes the live component's state, assigns it to `:tabs` within the parent live view's socket, and registers the hook:

```elixir
def init(socket) do
  socket
  |> assign(:tabs, %State{})
  |> attach_hook(:tabs_hook, :handle_params, fn params, uri, socket ->
      # TODO: implement
  end)
end
```

Next, we call the live component using the updated syntax:

```elixir
def mount(_params, _session, socket) do
  {:ok, socket |> Tabs.init()}
end

def render(assigns) do
  ~H"""
  <.live_component module={Tabs} id="tabs" tabs={@tabs}>
    <:tab label="Name" id={:name_tab}>Simon</:tab>
    <:tab label="Address" id={:address_tab}>742 Evergreen Terrace, Springfield</:tab>
  </.live_component>
  """
end
```

Since state changes are now fed from the outside, we also need to adapt the update function:

```elixir
def update(assigns, socket) do
  maybe_inner_state = Map.get(assigns, :state)
  tabs = make_tabs(assigns.tab)
  state = assigns.tabs
  # use the active tab if already there or the first slot
  active_id = assigns.tabs.active_id || hd(tabs).id

  updated_state = %State{
    state
    | maybe_inner_state: maybe_inner_state,
      tabs: tabs,
      active_id: active_id
  }

  {:ok, socket |> assign(:state, updated_state)}
end
```

To make the entire setup work with URI parameters, we need to parse and assign them in the registered hook and, later, use URI parameters for redirection in the `handle_event` function for the `change_tab` event. Let's examine the hook logic:

```elixir
fn params, uri, socket ->
  state = socket.assigns.tabs
  parsed_uri = URI.parse(uri)

  if Map.has_key?(params, "tabs") do
    tabs = Map.get(params, "tabs") |> String.to_existing_atom()
    next_state = %State{state | uri: parsed_uri, active_id: tabs}
    {:cont, socket |> assign(:tabs, next_state)}
  else
    next_state = %State{state | uri: parsed_uri}
    {:cont, socket |> assign(:tabs, next_state)}
  end
end
```


This function is registered in the `init` function. We check if there is a parameter with the key `"tabs"`. If present, we set the `active_id` field in the state to this value. Additionally, we introduce a new field called `uri` in the state to store the current URI. This is necessary to avoid interfering with the existing URI when redirecting in the event handler triggered by clicking on a tab link:


```elixir 
def handle_event("change_tab", %{"id" => raw_id}, socket) do
  uri = socket.assigns.state.uri
  new_uri = put_param(uri, "tabs", raw_id)
  {:noreply, socket |> push_patch(to: URI.to_string(new_uri))}
end

defp put_param(%URI{} = uri, key, value) do
  current_params = URI.decode_query(uri.query || "")
  new_params = Map.put(current_params, key, value)
  ((uri.path || "") <> "?" <> URI.encode_query(new_params)) |> URI.parse()
end
```

The `handle_event` function handles the "change_tab" event by extracting the current URI from the state and creating a new URI with the updated "tabs" parameter. We then use `push_patch` to navigate to the new URI.
The helper function `put_param` takes the current URI and inserts a new key-value pair without affecting existing parameters. In our case, we assign the ID of the active tab with the key `"tabs"`. We patch to the resulting URI. 

One oddity to note is that we need to have a `handle_params` function present in the parent live view for the hooks to work. We can simply add it and return the socket unaltered.

That's it! We have implemented our tabs view to work with URI parameters. Or have we?

> [Open code for parameter example](https://github.com/smoes/reusable-elixir-components/tree/b8106d95796f3c8c4e5f80015e86784061bc6962) on GitHub


## An evil twin

If we want to include multiple tabs components within a single live view, we will encounter issues. Since we hardcoded the key used to assign the live component's state in the live view's socket, we cannot have multiple independent components. If one component changes its active tab, the others will be affected as well. Let's address this by passing a key to the `init` function of the live component and storing it in an additional field in the state struct, called `:id`.

```elixir
def init(socket, opts \\ []) do
  id = Keyword.get(opts, :id, :tabs)
  id_str = Atom.to_string(id)

  socket
  |> assign(id, %State{id: id})
  |> attach_hook(:"#{id}_hook", :handle_params, fn params, uri, socket ->
     ...
  end)
end
```

It is important to consider the unique identifier (`id`) whenever we access a live component's state or work with URI parameters. This key serves as the identity of our component and allows us to handle multiple tabs components independently.

```elixir
def mount(_params, _session, socket) do
  next_socket = socket |> Tabs.init(id: :tabs_1) |> Tabs.init(id: :tabs_2)
  {:ok, next_socket}
end

def render(assigns) do
  ~H"""
  <div class="flex flex-col space-y-10">
    <.live_component module={Tabs} id="tabs_1" tabs={@tabs_1}>
      <:tab label="Name" id={:name_tab}>Simon</:tab>
      <:tab label="Address" id={:address_tab}>742 Evergreen Terrace, Springfield</:tab>
    </.live_component>
    <.live_component module={Tabs} id="tabs_2" tabs={@tabs_2}>
      <:tab label="Name" id={:name_tab}>Homer</:tab>
      <:tab label="Address" id={:address_tab}>742 Evergreen Terrace, Springfield</:tab>
    </.live_component>
  </div>
  """
end
```

> [Open code for tabs with identity](https://github.com/smoes/reusable-elixir-components/tree/a7a5e522490196d716f1cd2bce548e0aff3540b6) on GitHub


## Talk to me

To fulfill requirement 4, we can make the live component send messages to the parent live view when the state changes. To achieve this, we add a field called `:inform_parent?` to the state struct. This field can be optionally configured in the `init` function of the live component.

```elixir
 def init(socket, opts \\ []) do
    ...
    inform_parent? = Keyword.get(opts, :inform_parent?, false)

    socket
    |> assign(id, %State{id: id})
    |> assign(id, %State{id: id, inform_parent?: inform_parent?})
    ... 
```

We then use this flag to determine whether we should send a message to the parent live view:

```elixir
def handle_event("change_tab", %{"id" => raw_id}, socket) do
  ...
  if state.inform_parent? do
    send(self(), {:tab_changed, state.id, raw_id |> String.to_existing_atom()})
  end

  {:noreply, socket |> push_patch(to: URI.to_string(new_uri))}
end
```

The parent live view can handle this message by implementing the following callback:

```elixir
def handle_info({:tab_changed, tabs_id, active_tab}, socket) do 
  ...
end
```

> [Open code for the inform-parent example](https://github.com/smoes/reusable-elixir-components/tree/fc3b0c213a07624cfed57d79824d82f3d2cac98f) on GitHub


## Taking control

The last requirement we haven't fulfilled is the controllability from the outside (_requirement 4_). Throughout this blog post, we have modified our live component in a way that it is entirely controlled by URI parameters. We can leverage this to control the component from the outside. To achieve that, we implement the following function:

```elixir
def put_active(socket, tab, id \\ :tabs) do
  state = Map.get(socket.assigns, id)
  uri = put_param(state.uri, id |> Atom.to_string(), tab |> Atom.to_string())
  push_patch(socket, to: URI.to_string(uri))
end
```

This function accepts the live view's socket, the ID of the tab to be activated, and the ID of the tabs component. It then updates the URI with an additional parameter to control the tabs component. This function can be used in callbacks like `handle_event` or `handle_info` of the parent live view to change tabs from the outside and take control.

However, at this stage, the abstraction starts to leak. Although we hide the implementation details, it is still possible to override an existing patch in the socket. I haven't found a more elegant solution yet to address this concern.

> [Open code for the outside-controll example](https://github.com/smoes/reusable-elixir-components/tree/ffe2b9465d37e00b66ecc09b6fb1f8aebf77a75f) on GitHub

## Conclusion

In this lengthy blog post, we have developed a controllable, composable, and optionally chatty tabs component that effectively abstracts away the underlying complexity for developers. The beauty of this implementation is that once it's set up, we can effortlessly compose it with similar components. For example, if we have a pagination component designed in the same manner, we could easily create a tab view with pagination integrated:

```elixir
<.live_component module={Tabs} id="tabs" tabs={@tabs} state={@something}>
  <:tab label="Items 1" id={:items_tab} :let={state}>
    <.live_component module={Pagination} id="pagination" tabs={@pagination} items={state}>
      <:content :let={current_page_items}> ...  </:content>
    </.live_component>
  </:tab>
  <:tab ... />
</.live_component>
```

And the best part is that both the tabs component and the pagination component would seamlessly integrate and work with the features we have described throughout this blog post. Since, i have personally tested and verified this, I can assure you that it will work perfectly :)

The final code with specs and some documentation can be found on GitHub:

> [Open code with specs and doc](https://github.com/smoes/reusable-elixir-components/tree/449ce2499383d93fc41e9e1a1e2a832ca414e2b0) on GitHub


However, it is important to note that we don't have true composability in the sense that we can combine multiple live components and receive a new live component as a result. This limitation stems from the fact, that each component need the socket of the root live to register the hook. This requires a lot of manual work to plug components together. If I'll come up with a better version, I'll publish a follow up article.

Thanks for reading this blogpost until the end. Now give youself a break!


#### Disclosure

This article was proudly written without the help of any LLM.
