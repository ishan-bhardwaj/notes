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

### Storing data in a Component
```
export class UserComponent {
    selectedUser = USERS[0];
}
```
here -
- `selectedUser` is a `public` property. To mark it as `private` so that it can be used only within the class -
```
private selectedUser = USERS[0];
```
- We can then use the public properties in the html -
1. Using string interpolation -
```
<span>{{ selectedUser.name }}</span>
```
We can also use string interpolation to set value to another property -
```
<img src="{{ selectedUser.avatar }}" />
```
But better way is to use property binding.
2. Property binding -
```
<img [src]="'assets/users/' + selectedUser.avatar" />
```
This binds the `src` property of the underlying `<img>` DOM object to the value stored in `selectedUser.avatar`

> [!NOTE]
> Whilst it might look like you're binding the `src` attribute of the `<img>` tag, you're actually NOT doing that. Instead, property binding really targets the underlying DOM object property (in this case a property that's also called `src`) and binds that.
> The difference matters if you're trying to set attributes on elements dynamically. Attributes which don't have an equally-named underlying property.
> For example, when binding `ARIA` attributes, you can't target an underlying DOM object property.
> Since "Property Binding" wants to target properties (and not attributes), that can be a problem. That's why Angular offers a slight variation of the "Property Binding" syntax that does allow you to bind attributes to dynamic values.
> It looks like this - 
> ```
> <div 
>  role="progressbar" 
>  [attr.aria-valuenow]="currentVal" 
>  [attr.aria-valuemax]="maxVal">
> </div>
```
> By adding `attr` in front of the attribute name you want to bind dynamically, you're "telling" Angular that it shouldn't try to find a property with the specified name but instead bind the respective attribute - in the example above, the `aria-valuenow` and `aria-valuemax` attributes would be bound dynamically.
