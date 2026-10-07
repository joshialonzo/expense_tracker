# Angular 01 — Fundamentals: Standalone Components, Templates and Signals

## Why it matters
The JD says "React **or** Angular", so Angular is a nice-to-have topic: show you can read and build modern Angular and compare it with React. This series targets **modern Angular** (standalone components, signals, built-in control flow, `inject()`), which is the current default. Older material (NgModules, `*ngIf`, constructor injection) still exists in legacy code, so recognize it.

## Setup

```bash
npm i -g @angular/cli
ng new web-angular --routing --style=css --ssr=false
cd web-angular && ng serve        # http://localhost:4200
ng generate component features/expenses/expense-list
ng generate service core/expense-api
ng test        # unit tests (Karma/Jasmine or Jest/Vitest depending on version/config)
ng build       # production build in dist/
```
(Angular's tooling/default test runner changes between versions: check `ng version` and the docs for yours.)

## Component anatomy

```ts
// expense-list.component.ts
import { Component, computed, input, output, signal, ChangeDetectionStrategy } from "@angular/core";
import { CurrencyPipe, DatePipe } from "@angular/common";
import type { Expense } from "../types";

@Component({
  selector: "app-expense-list",
  standalone: true,                      // default in recent versions
  imports: [CurrencyPipe, DatePipe],
  changeDetection: ChangeDetectionStrategy.OnPush,
  templateUrl: "./expense-list.component.html",
})
export class ExpenseListComponent {
  // signal-based inputs/outputs
  expenses = input.required<Expense[]>();
  deleted = output<string>();

  filter = signal("");
  visible = computed(() =>
    this.expenses().filter(e => (e.description ?? "").toLowerCase().includes(this.filter().toLowerCase()))
  );
  totalCents = computed(() => this.visible().reduce((s, e) => s + e.amountCents, 0));

  onFilter(ev: Event) { this.filter.set((ev.target as HTMLInputElement).value); }
  remove(id: string) { this.deleted.emit(id); }
}
```

```html
<!-- expense-list.component.html -->
<input placeholder="Search…" [value]="filter()" (input)="onFilter($event)" />

@if (visible().length === 0) {
  <p>No expenses.</p>
} @else {
  <ul>
    @for (e of visible(); track e.id) {
      <li>
        {{ e.description ?? e.category }} — {{ e.amountCents / 100 | currency: e.currency }}
        <button type="button" (click)="remove(e.id)">Delete</button>
      </li>
    }
  </ul>
}
<strong>Total: {{ totalCents() / 100 | currency }}</strong>
```

Use in a parent:

```html
<app-expense-list [expenses]="expenses()" (deleted)="onDelete($event)" />
```

## Template syntax cheat sheet

| Syntax | Meaning |
|---|---|
| `{{ expr }}` | Interpolation |
| `[prop]="expr"` | Property binding (one-way, parent → child/DOM) |
| `(event)="handler($event)"` | Event binding (child/DOM → parent) |
| `[(ngModel)]="value"` | Two-way binding (sugar for the two above; needs `FormsModule`) |
| `@if` / `@else` / `@for (x of xs; track x.id)` / `@switch` / `@defer` | **Built-in control flow** (replaces `*ngIf`, `*ngFor`, `*ngSwitch`) |
| `{{ v \| pipe:arg }}` | Pipes (formatting: `currency`, `date`, `async`, custom) |
| `#ref` | Template reference variable |
| `[class.active]="cond"`, `[style.width.px]="w"` | Class/style bindings |

`track` is **required** in `@for` (like `key` in React). `@defer (on viewport) { <heavy-chart/> } @placeholder { … }` lazy-loads parts of a template.

## Signals (Angular's reactivity primitive)
- `signal(0)` writable state; read by calling `count()`. Update with `.set()` or `.update(v => v + 1)`.
- `computed(() => …)` derived and memoized, tracks dependencies automatically (no deps array!).
- `effect(() => …)` side effects when signals change (use sparingly: logging, syncing to localStorage).
- `input()`, `output()`, `model()` (two-way) replace `@Input()`/`@Output()` decorators.
- With signals + `OnPush`, change detection updates only what changed, and zone.js can eventually be dropped (zoneless).

```ts
count = signal(0);
double = computed(() => this.count() * 2);
increment() { this.count.update(c => c + 1); }
```

Compare with React: `useState` ≈ `signal`; `useMemo` ≈ `computed` (but automatic dependency tracking); `useEffect` ≈ `effect` (also automatic deps, avoid overuse).

## Change detection in two lines
Default: after any async event (zone.js patches events/timers/XHR) Angular checks the component tree. `OnPush`: component is checked only when an input reference changes, an event fires in it, a signal it reads changes, or you mark it. Prefer `OnPush` + signals/immutable data.

## Lifecycle hooks (most used)
`ngOnInit` (after inputs set; init logic), `ngOnChanges`, `ngAfterViewInit` (DOM available), `ngOnDestroy` (cleanup). Modern code often uses `inject(DestroyRef)` / `takeUntilDestroyed()` instead of manual unsubscribe.

## Exercise
Build the list above with add/delete/filter/total using signals only. Then convert `filter` into a `model()` and make a `<app-search-box [(value)]="filter">` child.

## Interview Q&A
- **What is a standalone component?** Self-contained: declares its own `imports` rather than belonging to an NgModule.
- **Signals vs RxJS?** Signals = synchronous state/derived values; RxJS = async streams/events over time (HTTP, debounce, websockets). They interoperate (`toSignal`, `toObservable`).
- **`@for` `track`?** Identity tracking for DOM reuse (like React keys).
- **`OnPush`?** Opt-in change detection by reference/signals; big perf win.
- **One-way vs two-way binding?** `[x]` + `(xChange)` = `[(x)]`.
