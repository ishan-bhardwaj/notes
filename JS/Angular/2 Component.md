# Components
- Components are created as classes that are instantiated by Angular itself.
- Decorators like `@Component` are used by Angular to add metadata & configuration to classes.

### Defining custom components - `header.component.ts`
```
import { Component } from '@angular/core';

@Component(
    selector: 'app-header',
    standalone: true,
    templateUrl: './header.component.html',
    styleUrls: ['./header.component.css']
)
export class HeaderComponent {}
```

where -
- `selector` is the name of the component that will be used as an html tag - `<app-header />`
- `templateUrl` defines the html content and `styleUrl` defines the css content for the component.
- `standalone` set to true mark the component as "standalone". There are "module"-based components also for which we'll have to set `standalone: false`

> [!NOTE]
> We can also use `styleUrl` property using which we can define single stylesheet -
> `styleUrl: './header.component.css'`

> [!NOTE]
> We can also define inline components like -
> ```
> template: '<p>Some content</p>'
> ```
> Similarly, we can define `styles` property to define multiple inline styles.
>
> But this is not recommended.

### Using the custom component inside another component
- `app.component.ts` -
```
import { HeaderComponent } from './header.component';

@Component(
    selector: 'app-root',
    standalone: true,
    imports: [HeaderComponent],
    templateUrl: './app.component.html',
    styleUrl: './app.component.css'
)
export class AppComponent {}
```
- Then we can use the component in `app.component.html` file - `<app-header />`

> [!WARNING]
> Make sure your assets are added to `angular.json` file.
> Look for `assets` key in the json and add path to your assets.

> [!TIP]
> We generally put components in separate directories. But we can also use angular-cli to generate components for us -
> `ng generate component <component-name>`
>
> or, in short
>
> `ng g c <component-name>`

