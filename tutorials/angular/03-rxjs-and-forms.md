# Angular 03 — RxJS Essentials and Reactive Forms

## RxJS in Angular
An **Observable** is a lazy stream of values over time (HTTP responses, form changes, route params, websockets). You **subscribe** to start it; **operators** transform it via `pipe`.

Observable vs Promise: lazy vs eager, many values vs one, cancellable (`unsubscribe`) vs not, rich operators.

### Operators you must know

| Operator | Use |
|---|---|
| `map`, `filter`, `tap` | transform / filter / side-effect |
| `switchMap` | map to an inner observable, **cancel previous** (search, route change) |
| `mergeMap` | run inner observables concurrently |
| `concatMap` | run inner observables sequentially (ordered writes) |
| `exhaustMap` | ignore new while one is running (prevent double-submit) |
| `debounceTime`, `distinctUntilChanged` | typeahead |
| `combineLatest`, `forkJoin` | combine streams (latest values / wait for all to complete) |
| `catchError`, `retry`, `timeout` | error handling |
| `takeUntil`, `take`, `takeUntilDestroyed` | completion/cleanup |
| `shareReplay(1)` | share one subscription among many consumers, cache last value |

### Typeahead search (classic)

```ts
searchCtrl = new FormControl("", { nonNullable: true });

results = toSignal(
  this.searchCtrl.valueChanges.pipe(
    debounceTime(300),
    map(q => q.trim()),
    distinctUntilChanged(),
    switchMap(q => q.length < 2 ? of([]) : this.api.search(q).pipe(catchError(() => of([])))),
  ),
  { initialValue: [] },
);
```
React equivalent: `useDebounced` + TanStack Query with an `AbortSignal` ([../react/02](../react/02-hooks-effects-and-refs.md)). In Angular, one RxJS pipeline replaces it.

### Memory leaks
Subscriptions that outlive the component leak. Fixes, in order of preference:
1. **`async` pipe** or **`toSignal`** (auto-unsubscribe).
2. `takeUntilDestroyed()` (call in injection context) or `takeUntilDestroyed(destroyRef)`.
3. Manual `unsubscribe` in `ngOnDestroy`.
HTTP observables complete by themselves, but long-lived streams (valueChanges, websockets, intervals) do not.

### Subjects
`Subject` (multicast, no initial), `BehaviorSubject` (holds current value, replays 1: used for simple stores), `ReplaySubject`. Expose as `asObservable()`; in new code prefer signals for local state.

## Reactive Forms (preferred over template-driven for anything serious)

```ts
import { FormBuilder, ReactiveFormsModule, Validators, AbstractControl, ValidationErrors } from "@angular/forms";

function notFuture(c: AbstractControl): ValidationErrors | null {
  return c.value && new Date(c.value) > new Date() ? { future: true } : null;
}

@Component({
  selector: "app-expense-form",
  standalone: true,
  imports: [ReactiveFormsModule],
  templateUrl: "./expense-form.component.html",
})
export class ExpenseFormComponent {
  private fb = inject(FormBuilder);
  private api = inject(ExpenseApi);
  categories = CATEGORIES;
  saving = signal(false);
  saved = output<Expense>();

  form = this.fb.nonNullable.group({
    amount:      [0, [Validators.required, Validators.min(0.01)]],
    category:    ["food", Validators.required],
    description: ["", Validators.maxLength(200)],
    date:        [today(), [Validators.required, notFuture]],
  });

  submit() {
    if (this.form.invalid) { this.form.markAllAsTouched(); return; }
    const v = this.form.getRawValue();
    this.saving.set(true);
    this.api.create({ amountCents: Math.round(v.amount * 100), currency: "USD", category: v.category, description: v.description, date: v.date })
      .pipe(finalize(() => this.saving.set(false)))
      .subscribe({ next: e => { this.saved.emit(e); this.form.reset(); } });
  }
}
```

```html
<form [formGroup]="form" (ngSubmit)="submit()" novalidate>
  <label for="amount">Amount</label>
  <input id="amount" type="number" step="0.01" formControlName="amount"
         [attr.aria-invalid]="form.controls.amount.invalid && form.controls.amount.touched" />
  @if (form.controls.amount.touched && form.controls.amount.errors; as err) {
    <span role="alert">
      @if (err['required']) { Amount is required } @else if (err['min']) { Must be greater than 0 }
    </span>
  }

  <select formControlName="category">
    @for (c of categories; track c) { <option [value]="c">{{ c }}</option> }
  </select>

  <input type="date" formControlName="date" />
  @if (form.controls.date.errors?.['future']) { <span role="alert">Date can't be in the future</span> }

  <button type="submit" [disabled]="saving()">Save</button>
</form>
```
Concepts: `FormControl`, `FormGroup`, `FormArray`; built-in and custom (sync/async) validators; cross-field validators on the group; `touched/dirty/pristine/valid/pending` state; typed forms (since v14) so `form.getRawValue()` is strongly typed. Client validation is UX; the server validates again ([../fullstack/01](../fullstack/01-rest-api-design.md)).

React Hook Form + Zod ([../react/04](../react/04-data-fetching-tanstack-query.md)) ↔ Angular Reactive Forms + validators.

## Exercise
Build the typeahead with `switchMap`, a reactive expense form with a custom validator and error messages, and a cross-field validator (`end >= start`) for a date-range filter. Prove with a test that rapid typing triggers only one HTTP call (use fake timers/`fakeAsync`).

## Interview Q&A
- **`switchMap` vs `mergeMap` vs `concatMap` vs `exhaustMap`?** Cancel previous / parallel / queue / ignore new.
- **How do you avoid memory leaks with RxJS?** `async` pipe/`toSignal`/`takeUntilDestroyed`.
- **Hot vs cold observable?** Cold = producer per subscriber (HTTP); hot = shared producer (events, Subjects); `shareReplay` makes cold → hot.
- **Template-driven vs reactive forms?** Reactive: explicit model in code, testable, scalable validation; template-driven: simple with `ngModel`.
- **How do you prevent double submission?** `exhaustMap` or disable while `saving`.
