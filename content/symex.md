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
User <span onMouseOver="popupText('Amelia was born in 1979...')">
    Amelia </span>
```

.box[
User <span onMouseOver="popupText('Amelia was born in 1979...')">
    Amelia </span>
]

---

template: string-injections

...but less so for `bio = "); alert('Boo!'); alert("`:

```html
User <span onMouseOver="popupText(''); alert('Boo!'); alert('')">
    Amelia
    </span>
```

which would inject code that would be executed (an XSS attack).

--


.box[
User <span onMouseOver="popupText(''); alert('Boo!'); alert('')">Amelia</span>
]

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


--

Of course, *all fragments* are decidable with finite variables!

---

# Aratha

![A flowchart depicting the execution flow in Aratha](content/images/aratha.png)

???

- JavaScript is instrumented by Jalangi2
- Essentially wrapping all variable accesses
- Best-effort: i.e. library calls etc may *just* return the concrete value

--


<div class="mermaid">
graph LR
        
        start("x = 'hello'")
        left("someFunction&#40;&#41;")
        right("someOtherFn&#40;&#41;")
        decision1{"x == 'bork'"}
        
        start --> decision1
        decision1 -- "PC1: x = 'bork'" --> left
        decision1 -- "PC2: x ≠ bork" --> right
        
</div>

???
- Get path condition, hand it to solver
- We need to "understand" what the code does!
- DSE: use these to generate inputs for max code coverage

---

# How They Modelled the JavaScript Semantics

- Approximations are allowed
- Variables become \\(\langle{}\mathit{type}, \mathit{string\: value}, \mathit{address}\rangle\\)
- The types are \\(\mathbb{T}=\left\\{\mathit{Null}, \mathit{Undef}, \mathit{Bool}, \mathit{Num}, \mathit{Str}, \mathit{Obj}\right\\}\\)
- Array writes: sequence of  \\(\langle O, \mathit{attribute} , \mathit{value} \rangle\\)
- Index writes and reads by time: \\(\mathit{read}(O, x, T) = i\\) means "at
  time \\(i\\), happening before time \\(T\\), the object \\(O\\) was given a
  value (which can be looked up).
- Deleting is writing `undefined`

???

- JS was designed in 10 days
- Objects are arrays ("dictionaries")
- The set of array indices is unbounded and arrays can alias!
- Encoding reads/writes is potentially expensive

---

# An Array Example

```javascript
y = O[a]; // read attribute a
O[a] = x; // write something to a
z = O[a]; // read the new value
```
- \\(j=read(O,a,i)\\): at \\(i\\), we read `O[a]`'s write from time \\(j\\)
- \\(write(O,a,x,i+1)\\): at \\(i + 1\\), we write `O[a] = x`
- \\(j'=read(O,a,i+1) \\): we read `O[a]`'s write from time \\(j'\\)
- \\(\mathit{type}(y) = \mathtt{PType[j] }\\): propagate the type etc of `y`
- \\(\mathit{sval}(y) = \mathtt{PSval[j] }\\)
- \\(\mathit{addr}(y) = \mathtt{PAddr[j]}\\)
- \\(\mathit{type}(z) = \mathtt{PType[j']} \\): propagate the type etc of `z`
- \\(\mathit{sval}(z) = \mathtt{PSval[j']} \\)
- \\(\mathit{addr}(z) = \mathtt{PAddr[j']}\\)

???

j' = i + 1

---

# G-Strings

![An example of a dashed string](content/images/dashed-strings.png)

> The name "dashed" comes from a graphical interpretation of \\(S = S_1^{l_1,
> u_1}, S_2^{l_2, u_2}, \ldots, S_2^{l_k,u_k}\\) where we imagine a block
> \\(S_i^{l_i,u_i}\\) as a continuous segment of length \\(l_i\\) followed by a
> dashed segment of length \\(u_i − l_i\\). The continuous segment indicates
> that exactly \\(l_i\\) characters of \\(S_i\\) *must* occur in each concrete
> string denoted by \\(S\\); the dashed segment indicates that \\(k\\)
> characters of \\(S_i\\), with \\(0 \leq k \leq u_i − l_i\\), *may* occur.
.footnote[Figure and quote from [A Novel Approach to String Constraint
Solving](https://doi.org/10.1007/978-3-319-66158-2_1).]

???
- Uses dashed strings
- each string is a concatenation of blocks
- each block has an upper and lower bound on length
- the interesting operation is membership
- the segments are {b,b}, [o^2, o^2], m, [!^0, !^3]

---

# The ExpoSE Test Suite

An example (`tests/regex/real_world/github.js`):
```javascript
var x = S$.symbol("X", '');

if (/^git(?:@|:\/\/)github\.com(?::|\/)([^\/]+\/[^\/]+)\.git$/
  .test(x)) {
	if (x.length > 0) throw 'Reachable';
	if (x.length == 0) throw 'Unreachable';
	if (x.indexOf('git') == -1) throw 'Unreachable';
	if (x.indexOf('@') == -1) throw 'Reachable';
}

```

--

.line3[![Point at `&x`](content/images/arrow.svg)]

Note that symbolic variables are annotated!

---

# Their Results

![Aratha benchmarks results](content/images/results.png)

- G-Strings has *higher* code coverage
- Timeouts make the results skewed
- Probably: Low % \\(\implies\\) imprecision in analysis
--

- *Are SMT solvers given bound*?
--
 🤷‍♀️
--

- All of ExpoSE's test file generate...no constraints???

---

# Summary

- DSE is whitebox fuzzing enhanced by constraint solvers
- Aratha: DSE engine that works with SMT *and* CP solvers
- CP solvers are now advanced enough to handle JS semantics
- Enables comparison
- Results seem *suspiciously* good
- Authors: "portfolio approach might be best"

---
class: center
background-image: url(https://source.unsplash.com/random?content_filter=high&orientation=squarish&featured=true)

.Title[FIN.]
