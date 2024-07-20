## React
- Javascript library for building user interfaces
- Creating new react project using Vite -
`npm create vite@latest <project_name>`
- Once created, run `npm install` to download required packages.
- To start development preview server (supports hot reloading) -
`npm run dev`

## Components
- Component name must start with upper-case letter
- Components are rendered as DOM nodes by React
- Components are functions -
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
```
import ReactDOM from "react-dom/client";
import App from "./App";

const entryPoint = document.getElementById("root");
ReactDOM.createRoot(entryPoint).render(<App />);
```
