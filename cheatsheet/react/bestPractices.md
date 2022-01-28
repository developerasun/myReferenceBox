# Best practices for using Typescript with React
by Christopher Diggins

There are numerous tools and tutorials to help developers start writing simple React applications with TypeScript. The best practices for using TypeScript in a larger React application are less clear, however.

This is especially the case when intergrating with an ecosystem of third party libraries used to address concerns such as: theming, styling, internationalization, logging, asynchronous communication, state-management, and form management.

At Clemex, we develop computational microscopy applications. We recently migrated a React front-end for one of our applications from JavaScript to TypeScript. Overall, we are very pleased with the end result. The consensus is that our codebase is now easier to understand and maintain.

That said, our transition was not without some challenges. This article dives into some of the challenges we faced and how we overcame them.

The challenges are primarily related to understanding the type signatures of the React API, and particularly those of higher order components. How can we resolve type errors correctly, while retaining the advantages of TypeScript?

This article attempts to address how to most effectively use TypeScript with React and the ecosystem of supporting libraries. We’ll also address some common areas of confusion.

## Finding type definitions for a library
A TypeScript program can easily import any JavaScript library. But without type declarations for the imported values and functions, we don’t get the full benefit of using TypeScript.

Luckily, TypeScript makes it easy to define type annotations for JavaScript libraries, in the form of type declaration files.

Only a few projects today offer TypeScript type definitions directly with the project. However, for many libraries you can usually find an up to date type-definition file in the @types organization namespace.

For example, if you look in the TypeScript React Template package.json, you can see that we use the following type-definition files:

"@types/jest": "^22.0.1","@types/node": "^9.3.0","@types/react": "^16.0.34","@types/react-dom": "^16.0.3","@types/redux-logger": "^3.0.5"
The only downside of using external type declarations is that it can be a bit annoying to track down bugs which are due to a versioning mismatch, or subtle bugs in type declaration files themselves. The type declaration files aren’t always supported by the original library authors.

## Compile-time validation of properties and state fields
One of the main advantages of using TypeScript in a React application is that the compiler (and the IDE, if configured correctly) can validate all of the necessary properties provided to a component.

It can also check that they have the correct type. This replaces the need for a run-time validation as provided by the prop-types library.

Here is a simple example of a component with two required properties:

import * as React from ‘react’;
export interface CounterDisplayProps {  value: number;  label: string;}
export class CounterDisplay extends React.PureComponent<CounterDisplayProps> {   render(): React.ReactNode {   return (     <div>       The value of {this.props.label} is {this.props.value}      </div>    );}

## Components as classes or functions
With React you can define a new component in two ways: as a function or as a class. The types of these two kinds of components are:

Component Classes :: React.ComponentClass<;P>
Stateless Functional Components (SFC) ::React.StatelessComponent<;P>
Component Classes
A class type is a new concept for developers from a C++/C#/Java background. A class has a special type, which is separate from the type of instance of a class. It is defined in terms of a constructor function. Understanding this is key to understanding type signatures and some of the type errors that may arise.

A ComponentClass is the type of a constructor function that returns an object which is an instance of a Component. With some details elided, the essence of the ComponentClass type definition is:

interface ComponentClass<P = {}> {  new (props: P, context?: any): Component<P, ComponentState>;}

## Stateless Components (SFC)
A StatelessComponent (also known as SFC) is a function that takes a properties object, optional list of children components, and optional context object. It returns either a ReactElement or null.

Despite what the name may suggest, a StatelessComponent does not have a relationship to a Component type.

A simplified version of the definition of the type of a StatelessComponent and the SFC alias is:

interface StatelessComponent<P = {}> {  (props: P & { children?: ReactNode }, context?: any):   ReactElement<any> | null;}
type SFC<P = {}> = StatelessComponent<P>;
Prior React 16, SFCs were quite slow. Apparently this has improved with React 16. However, due to the desire for consistent coding style in our code base, we continue to define components as classes.

## Pure and Non-Pure Components
There are two different types of Component: pure and non-pure.

The term ‘pure’ has a very specific meaning in the React framework, unrelated to the term in computer science.

A PureComponent is a component that provides a default implementation ofshouldComponentUpdate function (which does a shallow compare of this.stateand this.props).

Contrary to a common misconception, a StatelessComponent is not pure, and a PureComponent may have a state.

Stateful Components can (and should) derive from React.PureComponent
As stated above, a React component with state can still be considered a Pure component according to the vernacular of React. In fact, it is a good idea to derive components, which have an internal state, from React.PureComponent.

The following is based on Piotr Witek’s popular TypeScript guide, but with the following small modifications:

The setState function uses a callback to update state based on the previous state as per the React documentation.
We derive from React.PureComponent because it does not override the lifecycle functions
The State type is defined as a class so that it can have an initializer.
We don’t assign properties to local variables in the render function as it violates the DRY principle, and adds unnecessary lines of code.

import * as React from ‘react’;export interface StatefulCounterProps {  label: string;}
// By making state a class we can define default values.class StatefulCounterState {  readonly count: number = 0;};
// A stateful counter can be a React.PureComponentexport class StatefulCounter  extends React.PureComponent<StatefulCounterProps, StatefulCounterState>{  // Define  readonly state = new State();
  // Callbacks should be defined as readonly fields initialized with arrow functions, so you don’t have to bind them  // Note that setting the state based on previous state is done using a callback.  readonly handleIncrement = () => {    this.setState((prevState) => {       count: prevState.count + 1 } as StatefulCounterState);  }
  // We explicitly include the return type  render(): React.ReactNode {    return (      <div>        <span>{this.props.label}: {this.props.count} </span>        <button type=”button” onClick={this.handleIncrement}>           {`Increment`}        </button>      </div>     );  }}
  
## React Stateless Functional Components are not Pure Components
Despite a common misconception, stateless functional components (SFC) are not pure components, which means that they are rendered every time, regardless of whether the properties have changed or not.

## Typing higher-order components
Many libraries used with React applications provide functions that take a component definition and return a new component definition. These are called Higher-Order Components (or HOCs for short).

A higher-order component might return a StatelessComponent or a ComponentClass depending on how it is defined.

## The confusion of export default
A common pattern in JavaScript React applications is to define a component, with a particular name (say MyComponent) and keep it local to a module. Then, export by default the result of wrapping it with one or more HOC.

The anonymous component is imported throughout the application as MyComponent. This is misleading because the programmer is reusing the same name for two very different things!

To provide proper types, we need to realize that the component returned from a higher-order component is usually not the same type as the component defined in the file.

In our team we found it useful to provide names for both the defined component that is kept local to the file (e.g. MyComponentBase) and to explicitly name a constant with the exported component (e.g. export const MyComponent = injectIntl(MyComponentBase);).

In addition to being more explicit, this avoids the problem of aliasing the definition, which makes understanding and refactoring the code easier.

## HOCs that Inject Properties
The majority of HOCs inject properties into your component that do not need to be provided by the consumer of your component. Some examples that we use in our application include:

From material-ui: withStyles
From redux-form: reduxForm
From react-intl: injectIntl
From react-redux: connect
Inner, Outer, and Injected Properties
To better understand the relationship between the component returned from the HOC function and the component as it is defined, try this useful mental model:

Think of the properties expected to be provided by a client of the component as outer properties, and the entirety of the properties visible to the component definition (e.g. the properties used in the render function) as the inner properties. The difference between these two sets of properties are the injected properties.

## The type intersection operator
In TypeScript, we can combine types in the way we want to for properties using a type-level operator called the intersection operator (&). The intersection operator will combine the fields from one type with the fields from another type.

interface LabelProp {  label: string;}
interface ValueProp {  value: number;}
// Has both a label field and a value fieldtype LabeledValueProp = LabelProp & ValueProp;
For those of you familiar with set theory, you might be wondering why this isn’t considered a union operator. It is because it is an intersection of the sets of all possible values that satisfy the two type constraints.

## Defining properties for a wrapped component
When defining a component that will be wrapped with a higher-order component, we have to provide the inner properties to the base type (e.g. React.PureComponent<;P>).

However, we don’t want to define this all in a single exported interface, because these properties do not concern the client of the component: they only want the outer properties.

To minimize boilerplate and repetition, we opted to use the intersection operator, at the single point which we need to refer to inner properties type, which is when we pass it as a generic parameter to the base class.

interface MyProperties {  value: number;}
class MyComponentBase extends React.PureComponent<MyProperties & InjectedIntlProps> {  // Now has intl as a property  // ...}
export const MyComponent = injectIntl(MyComponentBase); // Has the type React.Component<MyProperties>;
The React-Redux connect function
The connect function of the React-Redux library is used to retrieve properties required by a component from the Redux store, and to map some of the callbacks to the dispatcher (which triggers actions which trigger updates to the store).

So, ignoring the optional merge function argument, we have potentially two arguments to connect:

mapStateToProps
mapDispatchToProps
Both of these functions provide their own subset of the inner properties to the component definition.

However, the type signature of connect is a special case because of the way the type was written. It can infer a type for the properties that are injected and also infer a type for the properties that are remaining.

This leaves us with two options:

We can split up the interface into the inner properties of mapStateToProps and another for the mapDispatchToProps.
We can let the type system infer the type for us.
In our case, we had to convert roughly 50 connected components from JavaScript to TypeScript.

They already had formal interfaces generated from the original PropTypes definition (thanks to an open-source tool we used from Lyft).

The value of separating each of these interfaces into outer properties, mapped state properties, and mapped dispatch properties did not seem to outweigh the cost.

In the end, using connect correctly allowed the clients to infer the types correctly. We are satisfied for now, but may revisit the choice.

Helping the React-Redux connect function infer types
The TypeScript type inference engine seems to need at times a delicate touch. The connect function seems to be one of those cases. Now hopefully this isn’t a case of cargo cult programming, but here are the steps we take to assure the compiler can work out the type.

We don’t provide a type to the mapStateToProps or mapDispatchToProps functions, we just let the compiler infer them.
We define both mapStateToProps and mapDispatchToProps as arrow functions assigned to const variables.
We use the connect as the outermost higher-order component.
We don’t combine multiple higher-order components using a compose function.
The properties that are connected to the store in mapStateToProps and mapDispatchToProps must not be declared as optional, otherwise you can get type errors in the inferred type.

## Final Words
In the end, we found that using TypeScript made our applications easier to understand. It helped deepen our understanding of React and the architecture of our own application.

Using TypeScript correctly within the context of additional libraries designed to extend React required additional effort, but is definitely worth it.

If you are just starting off with TypeScript in React, the following guides will be useful:

## Microsoft TypeScript React Starter
Microsoft’s TypeScript React Conversion Guide
Lyft’s React JavaScript to TypeScript Converter
After that I recommend reading through the following articles:

React Higher-Order Component Patterns in TypeScript by James Ravenscroft
Piotr Witek’s React-Redux TypeScript Guide
Acknowledgements
Many thanks to the members of Clemex team for their collaboration on this article, working together to figure out how to use TypeScript to its best potential in React applications, and developing the open-source TypeScript React Template project on GitHub.

## Reference
- [Best practices for using Typescript with React](https://www.freecodecamp.org/news/effective-use-of-typescript-with-react-3a1389b6072a/)