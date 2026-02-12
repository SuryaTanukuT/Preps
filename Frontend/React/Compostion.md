Composition means: build bigger UI by combining smaller components, instead of using inheritance.

In React, composition is the main design pattern.

1) Basic composition (Parent + Children)

function Page() {
  return (
    <>
      <Header />
      <Main />
      <Footer />
    </>
  );
}


2) Component as a “wrapper” using children

This is the most common composition pattern.

function Card({ children }) {
  return <div className="card">{children}</div>;
}

function App() {
  return (
    <Card>
      <h2>Title</h2>
      <p>Description</p>
    </Card>
  );
}

Why it’s powerful: Card doesn’t care what you put inside. It just provides layout/styling.

3) “Slots” (composition with named areas)

You can pass specific parts as props.

function Layout({ header, sidebar, content }) {
  return (
    <div>
      <div>{header}</div>
      <aside>{sidebar}</aside>
      <main>{content}</main>
    </div>
  );
}

<Layout
  header={<Header />}
  sidebar={<Menu />}
  content={<Dashboard />}
/>;

4) Higher-Order Component (HOC) as composition

Wrap a component to add behavior.

const withAuth = (Component) => (props) => {
  const isLoggedIn = true;
  return isLoggedIn ? <Component {...props} /> : <div>Please login</div>;
};


5) Render props (another composition style)

Pass a function that returns UI.

function DataFetcher({ render }) {
  const data = { name: "Surya" };
  return render(data);
}

<DataFetcher render={(data) => <h1>{data.name}</h1>} />;


(These days, hooks often replace render-props, but concept is still asked in interviews.)


Why composition is preferred over inheritance in React::
More flexible and reusable
Easier to test
Keeps components small + focused
Avoids deep inheritance hierarchies (React doesn’t encourage inheritance)

Interview one-liner::
React composition = “Combining components via props and children to reuse UI and behavior.”



