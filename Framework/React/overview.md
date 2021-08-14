# ReactJS overview

[ReactJS](https://reactjs.org/) is a JavaScript library for building user interfaces.

## Components and props

[Components](https://reactjs.org/docs/components-and-props.html) are the basic building block of ReactJS. Conceptually, they are like JavaScript functions. They accept arbitrary inputs `props` and return React elements describing what should appear on the screen.

The simplest way to define a component is to write a JavaScript function:

```javascript
function Welcome(props) {
    return <h1>Hello, {props.name}</h1>;
}
```

This function is a valid React component because it accepts a single `props` (which stands for properties) object argument with data and returns a React element. Such components are called `function components` because they are literally JavaScript functions.

Another way to define a component is to use an ES6 class:

```javascript
class Welcome extends React.Component {
    render() {
        return <h1>Hello, {this.props.name}</h1>;
    }
}
```

The above two components are equivalent from React's point of view.

## JSX

React components use [JSX](https://jsx.github.io/), a syntax extension to JavaScript. During the build process, the JSX code is transpiled to regular JavaScript (ES5) code.

The following two examples are equivalent:

```javascript
// JSX
const element = (
    <h1 className="greeting">
    Hello, world!
    </h1>
);

// Transpiled to createElement() call
const element = React.createElement(
  'h1',
  {className: 'greeting'},
  'Hello, world!'
);
```

## Elements

New React elements are created from component classes using the `createElement` function:

```javascript
React.createElement(
  type,
  [props],
  [...children]
)
```

This function takes three arguments:
- `type` can be either a tag name string (such as `div` or `span`), or a component class.
- `props` contains a list of attributes passed to the new element.
- `children` contains the child node(s) of the new element (which are additional React components).

## Safe by design

ReactJS implements security controls by design, for example, string variables in views are escaped automatically.

# References

- [Exploiting Script Injection Flaws in ReactJS Apps](https://medium.com/dailyjs/exploiting-script-injection-flaws-in-reactjs-883fb1fe36c1)
