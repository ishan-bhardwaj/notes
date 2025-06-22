## React

- Javascript library for building user interfaces
- Creating new react project using Vite -
`npm create vite@latest <project_name>`
- Once created, run `npm install` to download required packages.
- To start development preview server (supports hot reloading) - `npm run dev`

## Components

- Component name must start with upper-case letter.
- React "traverses" the component tree until it has only built-in components left - hence converting the component tree into DOM node tree.
- Components are functions that return JSX (JavaScript Syntax Extension).

**App.jsx** -
```
function Header() {
    return (<JSX Code>);
}

function App() {
    return (
        <div>
            <Header />
            <p> Hello World! </p>
        </div>
    )
}

export default App;
```

- Defining Root component -
**index.jsx** -
```
import ReactDOM from "react-dom/client";

import App from "./App";
import "./index.css";

const entryPoint = document.getElementById("root");
ReactDOM.createRoot(entryPoint).render(<App />);
```

where, `root` element is defined in the `index.html` file -
```
<body>
    <div id="root"></div>
    <script type="module" src="/src/index.jsx"></script>
</body>
```

- `ReactDOM` is responsible for rending the `App` component on UI - by injecting the `App` component into the `root`.

- `{}` is used to inject dynamic expression in JSX -
```
function showImage() {
    return (
        <img src={filePath}>
    )
}
```

> [!WARNING]
> if-statements, for-loops, function definitions are not allowed inside {}. Only the expressions that directly produce a value are allowed.

> [!TIP]
> Rendering images using `<img src=assets/img.png>` is not a good practise as the image can be lost during the project build process.
> Better way is to import the image - `import vehicleImg from './assets/vehicle.png'` 
> and then use it - `<img src={vehicleImg}>`
> This will include an auto-generated path that will also work once you deploy the React app on a server.


## Props

- "Custom HTML attributes" set on components.
- React merges all props into a single object and then props are passed to the component function as an argument.
```
<UserInfo 
    name="John"
    age={30}
    details={{username: 'John'}}
    hobbies=[['Cooking', 'Reading']]
/>
```

> [!NOTE]
> In `{34}`, curly braces are required to pass the value as a number.

- `UserInfo` component -

```
function UserInfo(props) {
    return (
        <p>Hello {props.name}!</p>
    )
}
```

