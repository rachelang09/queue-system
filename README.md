# Massage Chair Queue

A ticket queue for a shared massage chair. People take a number, choose how long they'll use the chair (15–30 minutes), and get an alert when it's their turn. The board shows the past, current and next numbers.

- `index.html` — single-screen version (e.g. a tablet next to the chair). Open it in a browser; the queue is saved in that browser only.
- `shared/massage-queue.html` — multi-device version, published as a claude.ai Artifact that uses its shared database so everyone can take a number from their own phone.

Run the single-screen version locally:

```
python -m http.server 8123
```

then open http://localhost:8123.
