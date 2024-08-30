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

- `{}` is used to inject dynamic expression in JSX -
```
function showImage() {
    return (
        <img src={filePath}>
    )
}
```

> [!TIP]
> Rendering images using `<img src=assets/img.png>` is not a good practise as the image can be lost during the project build process.
> Better way is to import the image - `import reactImg from 'assets/img.png'` 
> and then use it - `<img src={reactImg}>`
> This will include an auto-generated path that will also work once you deploy the React app on a server.

