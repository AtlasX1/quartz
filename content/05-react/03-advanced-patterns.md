# Advanced React Patterns: L4 Engineering Guide

## Part 1: Composition Patterns & Component Architecture

### 1.1 Render Props Pattern

**Render props** pass a function as a prop to share logic. The component calls this function to render UI, enabling logic reuse without wrapper components. This pattern inverts control: child component controls what parent renders.

```jsx
// Render props component
function MouseTracker({ render }) {
  const [position, setPosition] = React.useState({ x: 0, y: 0 });

  React.useEffect(() => {
    const handleMouseMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    window.addEventListener("mousemove", handleMouseMove);
    return () => window.removeEventListener("mousemove", handleMouseMove);
  }, []);

  return render(position);
}

// Usage
function App() {
  return (
    <MouseTracker
      render={({ x, y }) => (
        <div>
          Mouse at ({x}, {y})
        </div>
      )}
    />
  );
}

// Real-world: DataFetcher render prop
function DataFetcher({ url, render, renderError, renderLoading }) {
  const [state, setState] = React.useState({ data: null, loading: true, error: null });

  React.useEffect(() => {
    fetch(url)
      .then(r => r.json())
      .then(data => setState({ data, loading: false, error: null }))
      .catch(error => setState({ data: null, loading: false, error }));
  }, [url]);

  if (state.loading) return renderLoading?.() || <div>Loading...</div>;
  if (state.error) return renderError?.(state.error) || <div>Error</div>;
  return render(state.data);
}
```

### 1.2 Higher-Order Components (HOCs)

**HOCs** wrap components to add functionality. A HOC is a function that takes a component and returns an enhanced component. Enables cross-cutting concerns without modifying original components.

**Caution:** HOCs create wrapper components, adding DOM depth. They break method references and can cause key issues. Hooks often provide cleaner solutions.

```jsx
// HOC: withTheme
function withTheme(Component) {
  return function ThemedComponent(props) {
    const [theme, setTheme] = React.useState("light");
    
    return (
      <div className={`theme-${theme}`}>
        <Component {...props} theme={theme} setTheme={setTheme} />
      </div>
    );
  };
}

// HOC: withDataFetching
function withDataFetching(url) {
  return function WithDataFetching(Component) {
    return function DataFetchingComponent(props) {
      const [data, setData] = React.useState(null);
      const [loading, setLoading] = React.useState(true);
      const [error, setError] = React.useState(null);

      React.useEffect(() => {
        fetch(url)
          .then(r => r.json())
          .then(data => { setData(data); setLoading(false); })
          .catch(err => { setError(err); setLoading(false); });
      }, []);

      return (
        <Component
          {...props}
          data={data}
          loading={loading}
          error={error}
        />
      );
    };
  };
}

// Usage
const EnhancedList = withDataFetching("/api/items")(ItemList);
```

### 1.3 Compound Components Pattern

**Compound components** are a set of components that work together with implicit state sharing. Parent component manages state; children communicate through context or render props, enabling flexible composition.

```jsx
// Compound component: Accordion
function Accordion({ children }) {
  const [activeIndex, setActiveIndex] = React.useState(null);
  return (
    <AccordionContext.Provider value={{ activeIndex, setActiveIndex }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

function AccordionItem({ index, title, children }) {
  const { activeIndex, setActiveIndex } = React.useContext(AccordionContext);
  const isActive = activeIndex === index;

  return (
    <div className="accordion-item">
      <button onClick={() => setActiveIndex(isActive ? null : index)}>
        {title}
      </button>
      {isActive && <div className="accordion-content">{children}</div>}
    </div>
  );
}

const AccordionContext = React.createContext();

// Usage
<Accordion>
  <AccordionItem index={0} title="Section 1">Content 1</AccordionItem>
  <AccordionItem index={1} title="Section 2">Content 2</AccordionItem>
</Accordion>
```

---

## Part 2: Advanced Component Composition

### 2.1 Function-as-Child Pattern (FARC)

**Function-as-child** passes a function directly as child, enabling ultra-flexible component APIs. Child function receives state/methods from parent; parent renders what child function returns.

```jsx
// Function-as-child component
function Toggle({ children }) {
  const [isOpen, setIsOpen] = React.useState(false);
  return children({ isOpen, toggle: () => setIsOpen(!isOpen) });
}

// Usage
<Toggle>
  {({ isOpen, toggle }) => (
    <>
      <button onClick={toggle}>Toggle</button>
      {isOpen && <div>Content</div>}
    </>
  )}
</Toggle>

// Advanced: Query component with function-as-child
function Query({ query, children }) {
  const [data, setData] = React.useState(null);
  const [loading, setLoading] = React.useState(true);

  React.useEffect(() => {
    fetch(query)
      .then(r => r.json())
      .then(data => { setData(data); setLoading(false); });
  }, [query]);

  return children({ data, loading });
}
```

### 2.2 Component Polymorphism & `as` Prop

**Polymorphic components** change their underlying DOM element via an `as` prop. Common pattern for design systems: single component renders as `button`, `a`, `div`, etc., with proper semantics.

```jsx
// Polymorphic Button component
function Button({ as: Component = "button", children, ...props }) {
  return <Component {...props}>{children}</Component>;
}

// Usage
<Button>Submit</Button> {/* Renders <button> */}
<Button as="a" href="/home">Home</Button> {/* Renders <a> */}
<Button as="div">Clickable Div</Button> {/* Renders <div> */}

// With TypeScript for proper typing
interface ButtonProps<C extends React.ElementType> {
  as?: C;
  children: React.ReactNode;
}

function Button<C extends React.ElementType = "button">({
  as: Component = "button" as C,
  children,
  ...props
}: ButtonProps<C> & React.ComponentPropsWithoutRef<C>) {
  return <Component {...props}>{children}</Component>;
}
```

---

## Interview Questions

**Q1: Explain render props vs HOCs. When should you use each?**

Render props pass a function as prop; HOCs wrap components. Render props are more explicit (easier to debug) and compose naturally. HOCs add wrapper components (increasing DOM depth) but work with class components. Modern React prefers custom hooks (replaces both patterns). Use render props for explicit logic sharing; use HOCs for legacy code or cross-cutting concerns that need class component support.

**Q2: What problems do compound components solve?**

Compound components enable flexible APIs where related components work together. Children automatically have access to parent state via context without prop drilling. Enables intuitive component composition. Trade-off: children must be direct descendants (no wrapper divs between parent and child).

**Q3: Explain the `as` prop pattern and its benefits.**

The `as` prop enables polymorphic components: same component renders as different DOM elements. Benefits: code reuse, consistent styling, maintains semantic HTML (renders `button` or `a` appropriately). Essential for design systems where visual consistency matters across different semantic contexts.

**Q4: Why do hooks replace render props and HOCs?**

Custom hooks extract logic directly without adding components or wrapper DOM nodes. Hooks compose naturally without nesting; they're easier to debug and understand. However, render props/HOCs still useful in rare cases (e.g., HOCs for legacy class component patterns).

---

## Key Takeaways

1. **Render props enable logic reuse via functions** - Invert control to parent components
2. **HOCs wrap components for cross-cutting concerns** - Add wrapper DOM nodes; use sparingly
3. **Compound components enable flexible APIs** - Related components share state via context
4. **Function-as-child provides maximum flexibility** - Powerful but requires careful component design
5. **Polymorphic components (`as` prop) support multiple DOM elements** - Essential for design systems
6. **Custom hooks are preferred over render props/HOCs** - Simpler, no extra DOM nodes, more composable
7. **Choose patterns based on reusability needs** - Simple state sharing (hooks) vs complex cross-cutting concerns (HOCs)