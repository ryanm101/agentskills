---
description: An ordinary async/await refactor of a pasted Express route handler, with no binary, decompilation, or reverse-engineering content. Guards against re firing on any code-shaped request.
expected_outcome: The handler is refactored to async/await; re is never invoked.
max_turns: 12
allowed_tools: [Read, Glob, Grep, Skill]
---

Refactor this Express route handler to use async/await instead of callbacks:

```js
app.get('/orders/:id', function (req, res) {
  db.getOrder(req.params.id, function (err, order) {
    if (err) return res.status(500).send('error');
    res.json(order);
  });
});
```
