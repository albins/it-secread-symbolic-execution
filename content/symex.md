class: center
name: title
count: true

.Title[TITLE TBA]

<!-- .me[.grey[*by* **Nicholas Matsakis**]] -->

.citation[`https://github.com/albins/it-secread-symbolic-execution`]

---

# A Graph Example

<div class="mermaid">
graph LR
        A-->B
        B-->C
        C-->A
        D-->C
</div>

---

First part

--

Second part

$$x + y = 17$$

This is an inline integral: \\(\int_a^bf(x)dx\\)

---

name: a-template

# A Template With Some Code

```rust
let mut x: u32 = 22;
let y: &u32 = &x;
x += 1;
print(y);
```

---

template: a-template

.line2[![Point at `&x`](content/images/arrow.svg)]

---

# Thank you!

.center[.HugeEmoji[😍]]
