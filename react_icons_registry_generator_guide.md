# React Icons Registry Generator (LLM-Friendly Setup)

This guide explains how to generate and maintain an always-updated registry of React Icons for use with LLMs, ensuring correct icon names and pack mappings.

---

## 🚀 Goal

Create a machine-readable JSON file containing all icons from React Icons so an LLM can:

- Use correct icon names
- Avoid hallucinated icons
- Always stay updated when dependencies change

---

## 📦 Install Dependency

```bash
npm install react-icons
```

---

## 📁 Create Script File

Create a file in your project root (or `/scripts`) named:

```bash
generate-icons.mjs
```

---

## ⚙️ Script Code

Paste the following code into `generate-icons.mjs`:

```js
import fs from "fs";
import path from "path";

const ICONS_DIR = path.join(process.cwd(), "node_modules", "react-icons");

const packs = fs
  .readdirSync(ICONS_DIR)
  .filter((dir) => !dir.startsWith("."));

const registry = {};

for (const pack of packs) {
  const indexPath = path.join(ICONS_DIR, pack, "index.d.ts");

  if (!fs.existsSync(indexPath)) continue;

  const content = fs.readFileSync(indexPath, "utf-8");

  const matches = [...content.matchAll(/export declare const (\w+)/g)];

  registry[pack] = matches.map((m) => m[1]);
}

fs.writeFileSync(
  "icon-registry.json",
  JSON.stringify(registry, null, 2)
);

console.log("✅ Icon registry generated successfully");
```

---

## ▶️ Run Script

Run with Node:

```bash
node generate-icons.mjs
```

---

## 📄 Output

You will get a file:

```bash
icon-registry.json
```

Example structure:

```json
{
  "fa": ["FaHome", "FaUser", "FaBell"],
  "md": ["MdHome", "MdSettings"],
  "hi": ["HiOutlineBell"]
}
```

---

## 🔁 Keep It Updated

Whenever you update dependencies:

```bash
npm update react-icons
node generate-icons.mjs
```

Or add to package.json:

```json
{
  "scripts": {
    "update-icons": "node generate-icons.mjs"
  }
}
```

Run:

```bash
npm run update-icons
```

---

## 🧠 How to Use With LLMs

### Option 1: System Prompt Injection

Provide `icon-registry.json` to the model:

> Only use icons from this registry. Do not invent icon names.

---

### Option 2: RAG (Recommended)

Store icons in a database with metadata:

```json
{
  "name": "FaHome",
  "pack": "fa",
  "tags": ["home", "dashboard", "navigation"]
}
```

Then retrieve relevant icons during UI generation.

---

### Option 3: Tool Calling (Best)

Expose a function:

```ts
getIcon(name: string)
```

LLM selects icon → backend validates → returns safe result.

---

## 🔥 Pro Upgrade Ideas

To make this even more powerful:

- Add keyword tagging (home, settings, user, etc.)
- Build icon semantic search
- Embed icons in a vector DB for AI selection
- Cache registry for instant UI generation

---

## ✅ Result

You now have a fully deterministic icon system that:

- Prevents hallucinated icons
- Keeps UI consistent
- Automatically updates with dependencies
- Works perfectly with LLMs and AI UI generation

