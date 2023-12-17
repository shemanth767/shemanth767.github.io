---
title: React Design Patterns - Higher Order Component
date: 2023-12-17 10:00:00 +0530
categories: [JavaScript]
tags: []
toc: true
published: true
mermaid: false
---


## Intent

Higher Order Component (HOC) is an advanced design pattern in React that helps reuse similar logic across different components.
These are functions that take a component as input and wrap it with additional functionality. This pattern is also referred to
as the Decorator pattern.

## Problem

Consider a scenario where you have three different dropdown components in your React application.
These dropdowns share the same core functionalities, such as selecting an option and logging the usage of each
dropdown. However, they need to have distinct visual styles and appearances to match the design of parent components.

To take a basic example, we'll use the three dropdown styles below for reference. First one is Material UI styled, second
one is a basic HTML select dropdown, and the third one is Ant Design styled. All implementing the same core functionalities.

![](/assets/img/hoc-post/all-dropdowns.jpg)

To implement this, one approach is to create three separate components, each with identical state modification logic.
Despite performing the same logic, the visualization of the rendered elements in these components differ significantly.
Therefore, it is necessary to create three distinct components to accommodate the unique visual styles and appearances.

Below is the state modification logic repeated in all three components:
```ts
    // Constructor initializes the component state and initializes the necessary state variables. 
    constructor(props: {}) {
      super(props);
      this.state = {
        isOpen: false,
        selectedOption: null,
      };
    }

    // Switch whether the dropdown is hidden or shown.
    toggleDropdown = () => {
      this.setState((prevState) => ({
        isOpen: !prevState.isOpen,
      }));
    };

    // Select a specific option.
    selectOption = (option: string) => {
    // Log information about type of dropdown component. 
    console.log('Selected option in dropdown type: Ant Design');
      this.setState({
        selectedOption: option,
        isOpen: false,
      });
    };
```

However, this leads to code duplication and maintenance overhead. Additionally, the behavior of the components may
slightly differ, as seen in the example where the console.log value varies based on the component type. Repeating the
logic of these components, even though the UI rendering is different, introduces unnecessary complexity and increases
the chances of introducing bugs. A more efficient solution is needed to reuse the common logic while allowing for
customization of the UI appearance.

## Solution

We can use the Higher Order Component (HOC) pattern. The HOC will encapsulate the common logic for the dropdown
functionality and allow us to customize the UI appearance for each specific dropdown component. The HOC can take
the original dropdown component as input and return a new component with the additional functionality. This way,
we can reuse the logic across multiple components without duplicating code.

Here's an example of how the HOC can be implemented:

```tsx
const withDropdownLogic = (WrappedComponent: React.ComponentType<any>, type: string) => {
  return class extends React.Component<{}, {isOpen: boolean, selectedOption: string | null}> {
    constructor(props: {}) {
      super(props);
      this.state = {
        isOpen: false,
        selectedOption: null,
      };
    }

    toggleDropdown = () => {
      this.setState((prevState) => ({
        isOpen: !prevState.isOpen,
      }));
    };

    selectOption = (option: string) => {
      console.log('Selected option in dropdown type: ', type);
      this.setState({
        selectedOption: option,
        isOpen: false,
      });
    };

    render() {
      return (
        <WrappedComponent
          {...this.props}
          isOpen={this.state.isOpen}
          selectedOption={this.state.selectedOption}
          toggleDropdown={this.toggleDropdown}
          selectOption={this.selectOption}
        />
      );
    }
  };
};

export default withDropdownLogic;
```

Here's how it works: `withDropdownLogic` takes as input the component to be wrapped and the type of the component.
When invoked, it returns a wrapped component with additional state specific props passed to the wrapped component.
`WrappedComponent` no longer needs to bother about the state modification and needs to be concerned only about the
visualization.

Below is an example of how the HOC can be used:
```tsx
class MaterialDropdown extends React.Component<MaterialDropdownProps, MaterialDropdownState> {
    render() {
        const { isOpen, selectedOption, options, selectOption, toggleDropdown } = this.props;

        return (
            <Box sx={{ minWidth: 120 }}>
                <FormControl fullWidth>
                    <Select
                        value={selectedOption}
                        onChange={(e) => selectOption(e.target.value)}
                        open={isOpen}
                        onClose={toggleDropdown}
                        onOpen={toggleDropdown}
                    >
                        {options.map((option) => (
                            <MenuItem key={option} value={option}>
                                {option}
                            </MenuItem>
                        ))}
                    </Select>
                </FormControl>
            </Box>
        );
    }
}

export default withDropdownLogic(MaterialDropdown, "Material UI") as React.ComponentType<any>;
```

## Applicability

1. **Reusability:** Use HOCs when you have common functionality or state that needs to be shared across multiple components.
2. **Cross-Cutting Concerns:** HOCs are ideal for handling cross-cutting concerns such as authentication, logging, or error handling.
    Instead of duplicating this logic in multiple components, you can create a HOC that wraps the components and provides the
    necessary functionality.
3. **Prop Manipulation:** HOCs can be used to manipulate or enhance the props of a component. For example, you can use an HOC to inject
    additional props, modify existing props, or provide default values for props.

## Popular Examples

### Redux's connect()

Anyone who's dealt with global state management in React would have come across redux and its `connect`` method. This is a perfect example
of a HOC which injects global state as props into a wrapped component.

```ts
export default connect(
    (state) => {
        const { todos } = state
        return { todoList: todos.allIds }
    }
)(TodoList);
```

## React Router's withRouter()

Another example is React Router's `withRouter` method which attaches routing related logic and props (especially history),
```ts
import { withRouter } from 'react-router-dom';

const MyComponent = ({ history }) => {
  const handleClick = () => {
    history.push('/new-route');
  };

  return (
    <div>
      <button onClick={handleClick}>Go to New Route</button>
    </div>
  );
};

export default withRouter(MyComponent);
```


## Cons

1. **Component Wrapping:** HOCs wrap components, which can lead to a deeper component hierarchy. This can make it
    more challenging to debug and trace component relationships.
2. **Dependency on HOCs:** Components that heavily rely on HOCs can become tightly coupled to those HOCs. This
    can make it harder to reuse or refactor components without also modifying the HOCs.
3. **Naming Collisions:** Wrapping a component with multiple HOCs can potentially introduce naming collisions if
    they don't handle prop naming conflicts properly. This can result in unexpected behavior or errors.
