# Acceptance Criteria — Exercise 1 - Docker: Build and Run Locally

---

## Submission Checklist

The following must be present in the student's GitHub repository for the submission to be accepted.

---

### Required Files

| File | Required | Notes |
|---|---|---|
| `Dockerfile` | Yes | Must be present at project root |
| `.dockerignore` | Yes | Must exclude `node_modules` and `.git` at minimum |
| `screenshot.png` | Yes | Browser screenshot showing the app at `localhost:3000` |

---

### Dockerfile — Must Pass All

- [ ] Uses `FROM node:20-alpine` or a valid Node LTS base image (`node:18`, `node:20-alpine` etc.)
- [ ] Sets `WORKDIR` to a named directory (e.g. `/app`)
- [ ] Copies `package*.json` **before** copying source files (correct layer caching order)
- [ ] Runs `npm install` as a `RUN` instruction
- [ ] Copies remaining source with `COPY . .`
- [ ] Declares `EXPOSE 3000`
- [ ] Uses `CMD ["npm", "start"]` in exec form (JSON array, not shell string)

---

### `.dockerignore` — Must Pass All

- [ ] File exists at project root
- [ ] Contains `node_modules`
- [ ] Contains `.git`

---

### Screenshot — Must Pass All

- [ ] Screenshot is included in the submission
- [ ] Browser address bar clearly shows `localhost:3000` (or `localhost:PORT` if a different host port was used)
- [ ] The game app UI is visible and loaded (not a blank page or error screen)

---

### Build and Run — Assessor Verification

The assessor should be able to clone the repository and run the following without errors:

```bash
docker build -t gameapp:check .
docker run -p 3000:3000 gameapp:check
```

The app must load at `http://localhost:3000`.

---

## Not Accepted If

- Dockerfile is missing or empty
- `node_modules` is present inside the Docker image (indicates missing `.dockerignore`)
- Screenshot shows an error page, blank page, or does not show `localhost` in the address bar
- `CMD` is written in shell form (`CMD npm start`) — exec form is required
- `COPY package*.json` comes **after** `COPY . .` — incorrect ordering fails the caching criterion
- Image cannot be built from the submitted Dockerfile without modification

---

## Marking Notes

This exercise is assessed as **pass / not yet**. There is no partial credit. A submission either meets all criteria above or is returned with specific feedback for resubmission.
