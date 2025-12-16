# Form Handling in React: L4 Engineering Guide

## Part 1: Controlled & Uncontrolled Components

### 1.1 Controlled Components Pattern

**Controlled components** store form state in React state; input value is always synchronized with state. React is the "single source of truth" for input values. Every keystroke triggers state update.

**Advantages:** easy to validate in real-time, can conditionally enable/disable submit, can clear inputs programmatically.

**Disadvantage:** every keystroke triggers re-render (performance impact for large forms, but usually negligible).

```jsx
// Controlled component
function Form() {
  const [email, setEmail] = React.useState("");
  const [password, setPassword] = React.useState("");
  const [errors, setErrors] = React.useState({});

  const handleEmailChange = (e) => {
    const value = e.target.value;
    setEmail(value);
    
    // Real-time validation
    if (!value.includes("@")) {
      setErrors(prev => ({ ...prev, email: "Invalid email" }));
    } else {
      setErrors(prev => ({ ...prev, email: "" }));
    }
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    if (validate()) {
      // Submit form data
      console.log({ email, password });
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={handleEmailChange}
        aria-invalid={!!errors.email}
      />
      {errors.email && <span>{errors.email}</span>}

      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />

      <button type="submit" disabled={!email || !password}>
        Submit
      </button>
    </form>
  );
}
```

### 1.2 Uncontrolled Components & useRef

**Uncontrolled components** use DOM as source of truth; React doesn't manage input state. Access value via `ref` when needed (usually on submit).

**Use cases:** forms integrating with non-React code, file inputs (can't be controlled), large forms where controlled components cause performance issues.

```jsx
// Uncontrolled component
function Form() {
  const emailRef = React.useRef(null);
  const passwordRef = React.useRef(null);

  const handleSubmit = (e) => {
    e.preventDefault();
    const email = emailRef.current.value;
    const password = passwordRef.current.value;
    console.log({ email, password });
  };

  const handleReset = () => {
    emailRef.current.value = "";
    passwordRef.current.value = "";
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="email" ref={emailRef} defaultValue="" />
      <input type="password" ref={passwordRef} defaultValue="" />
      <button type="submit">Submit</button>
      <button type="button" onClick={handleReset}>Reset</button>
    </form>
  );
}

// File input (always uncontrolled)
function FileUpload() {
  const fileRef = React.useRef(null);

  const handleSubmit = (e) => {
    e.preventDefault();
    const file = fileRef.current.files[0];
    // Upload file
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="file" ref={fileRef} />
      <button type="submit">Upload</button>
    </form>
  );
}
```

---

## Part 2: Advanced Form Handling

### 2.1 Form Libraries & Validation

**Form libraries** (React Hook Form, Formik) handle complex form scenarios: multi-step forms, dynamic fields, validation, error management. They reduce boilerplate compared to manual controlled components.

**React Hook Form** emphasizes performance (doesn't re-render on every input change) via minimal state updates.

```jsx
// React Hook Form example
import { useForm } from "react-hook-form";

function Form() {
  const { register, handleSubmit, watch, formState: { errors } } = useForm({
    defaultValues: { email: "", password: "" }
  });

  const onSubmit = (data) => {
    console.log(data); // { email: "...", password: "..." }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        {...register("email", {
          required: "Email required",
          pattern: { value: /\S+@\S+/, message: "Invalid email" }
        })}
      />
      {errors.email && <span>{errors.email.message}</span>}

      <input
        type="password"
        {...register("password", {
          required: "Password required",
          minLength: { value: 8, message: "Min 8 chars" }
        })}
      />
      {errors.password && <span>{errors.password.message}</span>}

      <button type="submit">Submit</button>
    </form>
  );
}

// Formik example (alternative)
import { Formik, Form, Field, ErrorMessage } from "formik";
import * as Yup from "yup";

const validationSchema = Yup.object({
  email: Yup.string().email().required(),
  password: Yup.string().min(8).required()
});

function FormikForm() {
  return (
    <Formik
      initialValues={{ email: "", password: "" }}
      validationSchema={validationSchema}
      onSubmit={(values) => console.log(values)}
    >
      <Form>
        <Field name="email" type="email" />
        <ErrorMessage name="email" component="span" />

        <Field name="password" type="password" />
        <ErrorMessage name="password" component="span" />

        <button type="submit">Submit</button>
      </Form>
    </Formik>
  );
}
```

### 2.2 Complex Form Scenarios

**Multi-step forms** use state machine pattern to manage wizard flows. **Dynamic fields** allow adding/removing form sections. **Nested validation** validates dependent fields.

```jsx
// Multi-step form with useReducer
function MultiStepForm() {
  const [step, setStep] = React.useState(1);
  const [data, setData] = React.useState({
    personal: { name: "", email: "" },
    address: { street: "", city: "" },
    billing: { cardNumber: "", expiry: "" }
  });

  const updateData = (section, values) => {
    setData(prev => ({
      ...prev,
      [section]: { ...prev[section], ...values }
    }));
  };

  const handleNext = () => {
    if (validateStep(step)) {
      setStep(step + 1);
    }
  };

  const handleSubmit = () => {
    if (validateAll()) {
      // Submit all data
      console.log(data);
    }
  };

  return (
    <div>
      {step === 1 && (
        <PersonalInfo data={data.personal} onChange={(v) => updateData("personal", v)} />
      )}
      {step === 2 && (
        <AddressInfo data={data.address} onChange={(v) => updateData("address", v)} />
      )}
      {step === 3 && (
        <BillingInfo data={data.billing} onChange={(v) => updateData("billing", v)} />
      )}

      <button onClick={() => setStep(step - 1)} disabled={step === 1}>Back</button>
      {step < 3 && <button onClick={handleNext}>Next</button>}
      {step === 3 && <button onClick={handleSubmit}>Submit</button>}
    </div>
  );
}

// Dynamic fields (add/remove inputs)
function DynamicForm() {
  const [fields, setFields] = React.useState([{ id: 0, value: "" }]);

  const addField = () => {
    setFields([...fields, { id: Date.now(), value: "" }]);
  };

  const removeField = (id) => {
    setFields(fields.filter(f => f.id !== id));
  };

  const updateField = (id, value) => {
    setFields(fields.map(f => (f.id === id ? { ...f, value } : f)));
  };

  return (
    <div>
      {fields.map((field) => (
        <div key={field.id}>
          <input
            value={field.value}
            onChange={(e) => updateField(field.id, e.target.value)}
          />
          {fields.length > 1 && (
            <button onClick={() => removeField(field.id)}>Remove</button>
          )}
        </div>
      ))}
      <button onClick={addField}>Add Field</button>
    </div>
  );
}
```

---

## Interview Questions

**Q1: Explain controlled vs uncontrolled components. When would you use each?**

Controlled components store form state in React; input value is always synchronized. Uncontrolled components use DOM as source of truth; access value via ref. Use controlled for real-time validation, conditional enable/disable, programmatic control. Use uncontrolled for simpler forms, integrating non-React code, file inputs, or when controlled components cause performance issues.

**Q2: Why use form libraries instead of manual controlled components?**

Form libraries (React Hook Form, Formik) reduce boilerplate, handle validation, manage errors/touched fields, support dynamic fields. React Hook Form optimizes performance via minimal re-renders. Manual controlled components require more code and are error-prone for complex forms.

**Q3: How do you validate form fields across multiple steps?**

Store data from all steps in parent state or context. Validate only current step on "Next", validate all on "Submit". Use async validation (API calls) for server-side validation. Display errors per-step or at end depending on UX requirements.

**Q4: What's the performance implication of controlled components?**

Each keystroke triggers state update and re-render. For large forms with many fields, this can cause lag if re-renders are expensive. Solution: use form libraries that batch updates, use uncontrolled components with validation on blur/submit, or memoize expensive child components.

---

## Key Takeaways

1. **Controlled components: React manages state** - Easy validation, conditional submit, programmatic control
2. **Uncontrolled components: DOM is source of truth** - Simpler for basic forms, integrating non-React code
3. **File inputs are always uncontrolled** - Can't programmatically set file values for security reasons
4. **Form libraries reduce boilerplate** - React Hook Form, Formik handle validation, errors, dynamic fields
5. **React Hook Form optimizes performance** - Minimal re-renders, field-level updates
6. **Multi-step forms use state machines** - Manage step transitions, preserve data across steps
7. **Validate on blur vs on change** - Blur for less disruptive UX; change for real-time feedback
8. **Nested/dependent validation** - Validate fields based on other field values