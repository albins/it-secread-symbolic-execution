class: center
name: title
count: true

background-image: url(https://source.unsplash.com/random?content_filter=high&orientation=squarish&featured=true)
.Title[Constraint Programming for Dynamic Symbolic Execution of JavaScript]

.Subtitle[Detecting Security Holes with Constraint Solvers]

<!-- .me[.grey[*by* **Nicholas Matsakis**]] -->


.citation[Slides available at `https://github.com/albins/it-secread-symbolic-execution`].

---
layout: true
name: string-injections
# String Injections

```html
User <span onMouseOver="popupText('{{bio}}')">
  {{userName}}
</span>
```

---
template: string-injections

(This example is from [Chen et al., 2017](http://doi.org/10.1145/3158091))

---

template:string-injections
Ok if `bio = "Amelia was born in 1979..."`:
```html
User
<span onMouseOver="popupText('Amelia was born in 1979...')">
Amelia </span> </h1>
```
---

template: string-injections

...but less so for `bio = "); alert('Boo!'); alert("`:

```html
<span onMouseOver="popupText(''); alert('Boo!'); alert('')">
```

which would inject code that would be executed (an XSS attack).

---
layout: true
name: DSL
# (Dynamic) Symbolic Execution

---
template: DSL

> DSE collects the *constraints* ... encountered at conditional statements
> during concrete execution; then, a *constraint solver* ... is used to *detect
> alternative execution paths* by systematically negating the *path conditions*.
> This process is repeated until *all the feasible paths are covered* ...

— (Amadini et. al, emphasis mine)

--

TL;DR:

> Determine program flow by feeding conditionals to a constraint solver.

---

layout: true

---
template: string-injections

When do we have an injection? Define an *attack pattern* \\(P\\). Then we have:

--

\\[
x_1 = \mathtt{replaceAll}(temp, \mathtt{\\{\\{userName\\}\\}}, user)
\\]
\\[
\land{}\:  x_2 = \mathtt{replaceAll}(x1, \mathtt{\\{\\{bio\\}\\}}, bio)
\\]
\\[
\land\: x_2 \in P
\\]

--

In this case, \\(P\\) could be e.g. a regular expression identifying an
injection.


---
# CP vs SMT

| CP                                      | SMT                                          |
| --------------------------------------- | -------------------------------------------- |
| Bounded variables                       | *Un*bounded variables                        |
| Planning/scheduling                     | Verification                                 |
| Discrete search                         | SAT solver + arbitrary logic(s)              |
| Branching ("try things")                |                                              |
| Remove infeasible values (*propagate*)  | Theories handle constraints                  |
| Complete, terminating                   | Who knows                                    |
| Problem formulation = model             | Problem = formula                            |
| Result = solution (store)               | Result = model                               |

---

# Many Interesting Problems are Undecidable

- Symbolic execution in general
- 3 string variables, concatenation (\\(s_1 \cdot c\\)), and equality (\\(s_1 = s_2\\))
- `replaceAll` with variable patterns (\\(\mathtt{replaceAll}(s_1, s_2, s_3)\\)) [1]
- `replaceAll` with constants and string length (\\(\|s_1\|\\)) [1]
- string equations with string to/from integer conversions [2]

.footnote[
[1]: [What is decidable about String Constraints with the ReplaceAll
function?](http://doi.org/10.1145/3158091)

[2]: [The Satisfiability
of Word Equations: Decidable and Undecidable
Theories](http://doi.org/10.1007/978-3-030-00250-3_2)]



--

We handle this by solving *decidable fragments*.

---

# How They Modelled the JavaScript Semantics

---

# G-Strings

---

# The ExpoSE Benchmarks

---

# Their Results

---
class: center
background-image: url(https://source.unsplash.com/random?content_filter=high&orientation=squarish&featured=true)

.Title[FIN.]
