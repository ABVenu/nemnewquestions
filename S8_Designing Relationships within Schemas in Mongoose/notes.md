### **Notes on Testing CRUD Operations in Express + Mongoose with Protected Routes**

#### **1. Setting Up the Environment**

- Use `.env.testing` to configure environment variables specific to the testing environment (e.g., database URL).
- Tools used:
  - **Jest**: JavaScript testing framework.
  - **Supertest**: HTTP assertions for testing APIs.
  - **Mongoose**: For MongoDB integration.

#### **2. Sample Express App (Recipe App)**

This app has two models: `User` and `Recipe`.

**app.js (basic Express setup)**

```javascript
const express = require("express");
const mongoose = require("mongoose");
const recipeRoutes = require("./routes/recipeRoutes");

const app = express();
app.use(express.json());

// Route setup
app.use("/api/recipes", recipeRoutes);

module.exports = app;
```

#### **3. Protected Routes**

Protected routes will require a token. Use `jsonwebtoken` for authentication:

**middleware/auth.js**

```javascript
const jwt = require("jsonwebtoken");

module.exports = (req, res, next) => {
  const token = req.header("Authorization");
  if (!token) return res.status(401).send("Access denied. No token provided.");

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (ex) {
    res.status(400).send("Invalid token.");
  }
};
```

---

### **4. CRUD Routes Example**

**routes/recipeRoutes.js**

```javascript
const express = require("express");
const router = express.Router();
const Recipe = require("../models/Recipe");
const auth = require("../middleware/auth");

// Create a new recipe (Protected)
router.post("/", auth, async (req, res) => {
  try {
    const recipe = new Recipe({ ...req.body, user: req.user._id });
    await recipe.save();
    res.status(201).send(recipe);
  } catch (error) {
    res.status(500).send(error);
  }
});

// Get all recipes (Protected)
router.get("/", auth, async (req, res) => {
  try {
    const recipes = await Recipe.find({ user: req.user._id });
    res.send(recipes);
  } catch (error) {
    res.status(500).send(error);
  }
});

module.exports = router;
```

**models/Recipe.js**

```javascript
const mongoose = require("mongoose");

const recipeSchema = new mongoose.Schema({
  name: { type: String, required: true },
  ingredients: { type: [String], required: true },
  user: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
});

module.exports = mongoose.model("Recipe", recipeSchema);
```

---

### **5. Setting Up `.env.testing`**

Create a `.env.testing` file for the testing environment:

```plaintext
MONGO_URI=mongodb://localhost:27017/recipe-app-test
JWT_SECRET=testsecret
```

---

### **6. Writing Tests for CRUD Operations with Protected Routes**

**test/recipeRoutes.test.js**

```javascript
const request = require("supertest");
const mongoose = require("mongoose");
const app = require("../app");
const Recipe = require("../models/Recipe");
const jwt = require("jsonwebtoken");

const userId = new mongoose.Types.ObjectId().toHexString();
const token = jwt.sign({ _id: userId }, process.env.JWT_SECRET);

beforeAll(async () => {
  await mongoose.connect(process.env.MONGO_URI);
});

afterEach(async () => {
  await Recipe.deleteMany();
});

afterAll(async () => {
  await mongoose.connection.close();
});

describe("Recipe CRUD Operations", () => {
  it("should create a new recipe", async () => {
    const response = await request(app)
      .post("/api/recipes")
      .set("Authorization", token)
      .send({
        name: "Test Recipe",
        ingredients: ["Flour", "Sugar"],
      });
    expect(response.status).toBe(201);
    expect(response.body.name).toBe("Test Recipe");
  });

  it("should get all recipes for the user", async () => {
    await new Recipe({
      name: "Recipe 1",
      ingredients: ["A"],
      user: userId,
    }).save();
    const response = await request(app)
      .get("/api/recipes")
      .set("Authorization", token);
    expect(response.status).toBe(200);
    expect(response.body.length).toBe(1);
  });

  it("should return 401 if no token is provided", async () => {
    const response = await request(app).get("/api/recipes");
    expect(response.status).toBe(401);
  });
});
```

### **8. Running the Tests Locally**

1. Ensure MongoDB is running on your system.
2. Run the tests using:
   ```bash
   npm test
   ```

---

### **7. Pushing to Github and setting up CI/CD Pipline**

- Once Code is pushed
- Setup a CI/CD pipeline by creating a Github Action, Please select and Configure NodeJS action
- Once you set up, a `yml` file will be created, now add data present in the env as Key, whose value will be added under `secrets` in the settings
- the env used should be added like

```
 - run: npm test
      env:
         NODE_ENV: test
         MONGO_URL: ${{ secrets.MONGO_URL }}
         SECRET_KEY: ${{ secrets.SECRET_KEY }}
```

- refer the following sample yml file

`.github/workflows/nodejs.yml`:

```yaml
name: Node.js CI

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18.x, 20.x, 22.x]
        # See supported Node.js release schedule at https://nodejs.org/en/about/releases/

    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: "npm"
      - run: npm ci
      - run: npm run build --if-present
      - run: npm test
        env:
          NODE_ENV: test
          MONGO_URL: ${{ secrets.MONGO_URL }}
          SECRET_KEY: ${{ secrets.SECRET_KEY }}
```

---

### **8. Now Add Secrets in GitHub**

- These secrets are the data you mentioned in the .env of your file

1. Go to your GitHub repository.
2. Click on **Settings**.
3. Then in left side bar, eelect **Secrets and Variables > Actions**.
4. Click **New Repository Secret**.
5. Add your secrets (e.g., `JWT_SECRET`, `MONGO_URI`) and save.
6. Please add Mongo Atlas/Cloud Url under secrets

- once secrets are added,then again click on actions and watch carefully that your code is been tested by Github Server
- Hence we have set a CI/CD pipline which tests the code pushed into cloud and keeps ready for deployment

---

### **Summary**

- Use `Jest` and `Supertest` to test CRUD operations.
- Protect routes using `jsonwebtoken`.
- Configure GitHub Actions to automate testing on code push.
- Add repository secrets to securely manage sensitive information.
