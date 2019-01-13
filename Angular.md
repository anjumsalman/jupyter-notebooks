## Modules
Collection of related components, services, directives sharinga common scope.

To define a module we create a class and decorate it with `@NgModule`.

Every app has atleast one module *root module*  called *App Module*.

### Module Properties
- **declarations**: components, directives or pipes
- **exports**: subset of declarations which can be used in other modules
- **imports**: list of modules whose classes will be used in this module
- **providers**: 

```ts
@NgModule({
  imports: [
    CommonModule,
    HttpModule,
    MiscModule
  ],
  declarations: [ProductTileComponent, ProductDetailsComponent],
  exports: [ProductTileComponent, ProductDetailsComponent]
})
```

## Components
Components contain logic of a view. A component and its template completes a view.

To create a component, decorate class with `@Component`.

### Component Metadata
- **selector**: HTML tag name for this component
- **templateUrl**: path to html template file
- **styleUrls**: list of stylesheets
- **providers**: list of services this component requires

### Component Template Basics
A component's template is plain html + some `template syntax`. Some of the things that are part of template syntax are:
- data binding
- pipes
- directives
- other child components

## Template Syntax
### Interpolation
Consider a simple component
```ts
\\ skipped few lines
export class AppComponent{
  title = 'App title';
}
```
```html
<h1>{{title}}</h1>
```
Using the above syntax, Angular pulls the value of title from the component and inserts that value in h1 tag. Whenever title changes, those changes are reflected automatically (but it needs some asynchronous event such as keystroke).  
  
The text between {{ }} is called `template expression`. Angular first evaluates it and then converts it to string. A template expression can
- contain component variable
- component methods
- math expression
- cannot contain expressions that have side effects such as new, =, ++, +=, etc 
- cannot refer to anything in global namespace such as window or document 
  
The `expression context` by default is the component. So when you write `{{title}}` it assumes that title is a property of component. Template expression can also contain properties of `template input variable` or `template reference variable`.
```html
<div *ngFor="let city of cities">{{city.name}}</div>
<input #nameInput> {{nameInput.value}}
```
### Template Statement
Template statement respond to events and are generally enclosed within `""`.
```html
<button (click)="deleteCity()">Delete</button>
```
Template statements obey rules similar to that of template expressions:
- no new, =, ++, +=, etc
- can't refer to window or document
  
`Template statement context` is typically the component. But again we can also use template input variables and reference variables.

### Binding
**From component to DOM**:
```html
<h1>{{title}}</h1>
<child-component [property]="value"></child-component>
```
**From DOM to component**:
```html
<button (click)="delete($event)">Delete</button>
```
**Two-way**:
```html
<input [(ngModel)]="student">
```
Always remember that `HTML Attributes` and `DOM Properties` are different. Some things to note:
- html attribute may have 1:1 mapping with dom property
  - example: id
- html attribute may not have corresponding dom property
  - example: colspan
- dom property may not have corresponding html attribute
  - example: textContent
- some dom properties appear to map
  - example: disabled

**Property binding**: set property of a view element.
```html
<img [src]="cityPhotoUrl">
```
The text within " " is template expression.
```html
<slideshow [autoScroll]="scollEnabled"></slideshow>
```
Slideshow in this case is a custom component.