`useForm` usually refers to **React Hook Form’s** hook (`react-hook-form`). It gives you everything to **register inputs**, **validate**, **submit**, and read **form state** with minimal re-renders. ([react-hook-form.com][1])

---

## What `useForm()` returns (core pieces)

From `useForm()` you commonly use: ([react-hook-form.com][1])

* **`register(name, rules)`** → connect an input to the form + add validation
* **`handleSubmit(onValid, onInvalid?)`** → submit handler wrapper
* **`formState`** → `errors`, `isSubmitting`, `isDirty`, `dirtyFields`, `touchedFields`, etc. ([react-hook-form.com][2])
* **`watch()`** → read field values as they change
* **`setValue`, `getValues`, `reset`, `trigger`, `setError`, `clearErrors`** → programmatic control

---

## Basic example (register + validation + submit)

```jsx
import { useForm } from "react-hook-form";

export default function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm({
    mode: "onSubmit", // validate on submit by default
    defaultValues: { email: "", password: "" },
  });

  const onSubmit = async (values) => {
    // values: { email, password }
    await fakeLogin(values);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <label>Email</label>
      <input
        {...register("email", {
          required: "Email is required",
          pattern: { value: /^\S+@\S+$/, message: "Invalid email" },
        })}
      />
      {errors.email && <p>{errors.email.message}</p>}

      <label>Password</label>
      <input
        type="password"
        {...register("password", {
          required: "Password is required",
          minLength: { value: 6, message: "Min 6 chars" },
        })}
      />
      {errors.password && <p>{errors.password.message}</p>}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Logging in..." : "Login"}
      </button>
    </form>
  );
}

async function fakeLogin() {
  await new Promise((r) => setTimeout(r, 500));
}
```

Docs for `register` + `useForm` basics: ([react-hook-form.com][3])

---

## Validation modes (when errors show)

React Hook Form supports modes like `onSubmit`, `onChange`, `onBlur`, `onTouched`, `all`. This affects when `errors` and `isValid` update. ([react-hook-form.com][1])

Practical pick:

* **`onSubmit`**: simplest UX
* **`onBlur`**: show errors after leaving a field
* **`onChange`**: immediate validation (can feel noisy)

---

## Using controlled components (with `Controller`)

For inputs that don’t work well with `register` (e.g., React Select, MUI components), use **`Controller`**. ([react-hook-form.com][4])

```jsx
import { Controller, useForm } from "react-hook-form";

function Profile() {
  const { control, handleSubmit } = useForm({ defaultValues: { country: "" } });

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <Controller
        name="country"
        control={control}
        render={({ field }) => (
          <select {...field}>
            <option value="">Select</option>
            <option value="IN">India</option>
            <option value="DE">Germany</option>
          </select>
        )}
      />
      <button type="submit">Save</button>
    </form>
  );
}
```

---

## Dynamic fields (arrays) with `useFieldArray`

For “Add more” patterns, use **`useFieldArray`**. ([react-hook-form.com][5])

```jsx
import { useForm, useFieldArray } from "react-hook-form";

function EmailsForm() {
  const { control, register, handleSubmit } = useForm({
    defaultValues: { emails: [{ value: "" }] },
  });

  const { fields, append, remove } = useFieldArray({ control, name: "emails" });

  return (
    <form onSubmit={handleSubmit(console.log)}>
      {fields.map((f, idx) => (
        <div key={f.id}>
          <input {...register(`emails.${idx}.value`, { required: true })} />
          <button type="button" onClick={() => remove(idx)}>Remove</button>
        </div>
      ))}
      <button type="button" onClick={() => append({ value: "" })}>Add</button>
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## Best practices (interview points)

* Provide **`defaultValues`** up front so RHF can correctly track `dirtyFields`/`touchedFields`. ([react-hook-form.com][6])
* Prefer **uncontrolled inputs + `register`** for best performance; use `Controller` only when needed. ([react-hook-form.com][7])
* Use `formState` flags (`isSubmitting`, `isDirty`, `errors`) to drive UI and prevent double submit. ([react-hook-form.com][2])
* For large forms, RHF patterns like `useFieldArray` are designed to keep UX responsive. ([react-hook-form.com][5])

---

## 20–30 sec interview answer

`useForm` from React Hook Form sets up a form controller that registers fields, validates them, and exposes `handleSubmit` plus `formState` like `errors` and `isSubmitting`. It favors uncontrolled inputs for performance, supports controlled components through `Controller`, and handles dynamic arrays with `useFieldArray`. ([react-hook-form.com][1])

[1]: https://react-hook-form.com/docs/useform?utm_source=chatgpt.com "useForm"
[2]: https://react-hook-form.com/docs/useform/formstate?utm_source=chatgpt.com "formState"
[3]: https://react-hook-form.com/get-started?utm_source=chatgpt.com "Get Started"
[4]: https://react-hook-form.com/docs/usecontroller/controller?utm_source=chatgpt.com "Controller"
[5]: https://react-hook-form.com/docs/usefieldarray?utm_source=chatgpt.com "useFieldArray"
[6]: https://react-hook-form.com/docs/useformstate?utm_source=chatgpt.com "useFormState"
[7]: https://www.react-hook-form.com/api/useform/register/?utm_source=chatgpt.com "useForm - register"
