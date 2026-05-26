# Recipe Site V1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first usable single-user recipe website with ingredients, conversions, nested recipes, stock, planned recipes, and computed shopping shortages.

**Architecture:** FastAPI serves Jinja-rendered pages backed by SQLite. SQLAlchemy models hold the relational data, while focused service modules own conversion math, recipe expansion, and shopping-list calculation. The UI is server-rendered with explicit dark-mode CSS and small JavaScript helpers for dynamic form rows and scaling controls.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.x, SQLite, Jinja2, pytest, Docker.

---

## File Structure

- Create `pyproject.toml`: package metadata, dependencies, pytest config, and lint-friendly defaults.
- Create `Dockerfile`: local container for the FastAPI app.
- Create `docker-compose.yml`: app service with persistent SQLite data volume.
- Create `app/__init__.py`: package marker.
- Create `app/main.py`: FastAPI app construction, routes registration, static/templates setup.
- Create `app/db.py`: SQLAlchemy engine/session setup and database initialization.
- Create `app/models.py`: SQLAlchemy tables for ingredients, conversions, recipes, recipe items, stock, and planned recipes.
- Create `app/schemas.py`: dataclasses/enums used by service-layer tests and route handlers.
- Create `app/services/units.py`: controlled units, global weight conversions, ingredient-specific conversions, formatting.
- Create `app/services/recipes.py`: recipe scaling, tree expansion, cycle detection, subtree/full-tree totals.
- Create `app/services/planning.py`: cart expansion, stock subtraction, shopping shortage formatting.
- Create `app/routes/recipes.py`: recipe list/detail/create/edit endpoints.
- Create `app/routes/ingredients.py`: ingredient and conversion create/edit endpoints.
- Create `app/routes/stock.py`: stock list/create/update endpoints.
- Create `app/routes/cart.py`: planned recipe list/create/update/delete endpoints.
- Create `app/routes/shopping.py`: computed To Buy endpoint.
- Create `app/templates/base.html`: shared page shell and navigation.
- Create `app/templates/recipes/*.html`: recipe list, detail, and form templates.
- Create `app/templates/ingredients/*.html`: ingredient/conversion list and form templates.
- Create `app/templates/stock/index.html`: stock management page.
- Create `app/templates/cart/index.html`: planned recipes page.
- Create `app/templates/shopping/index.html`: To Buy page.
- Create `app/static/styles.css`: explicit high-contrast dark theme.
- Create `app/static/app.js`: dynamic row helpers and basic client-side UI behavior.
- Create `tests/conftest.py`: isolated in-memory test database/session fixtures.
- Create `tests/test_units.py`: conversion tests.
- Create `tests/test_recipes.py`: scaling, nested recipe, rollup, and cycle tests.
- Create `tests/test_planning.py`: stock subtraction and shopping display tests.
- Create `tests/test_pages.py`: FastAPI page/form smoke tests.
- Modify `README.md`: local setup, Docker usage, test commands.

---

### Task 1: Project Skeleton And App Health

**Files:**
- Create: `pyproject.toml`
- Create: `app/__init__.py`
- Create: `app/main.py`
- Create: `app/templates/base.html`
- Create: `app/static/styles.css`
- Create: `tests/test_pages.py`

- [ ] **Step 1: Write the failing health/page test**

Create `tests/test_pages.py`:

```python
from fastapi.testclient import TestClient

from app.main import create_app


def test_home_redirects_to_recipes():
    client = TestClient(create_app())

    response = client.get("/", follow_redirects=False)

    assert response.status_code == 303
    assert response.headers["location"] == "/recipes"


def test_recipes_page_renders_empty_state():
    client = TestClient(create_app())

    response = client.get("/recipes")

    assert response.status_code == 200
    assert "Recipes" in response.text
    assert "No recipes yet" in response.text
```

- [ ] **Step 2: Add project dependencies**

Create `pyproject.toml`:

```toml
[project]
name = "recipessite"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
  "fastapi>=0.111",
  "jinja2>=3.1",
  "python-multipart>=0.0.9",
  "sqlalchemy>=2.0",
  "uvicorn[standard]>=0.30",
]

[project.optional-dependencies]
dev = [
  "httpx>=0.27",
  "pytest>=8.2",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]
```

- [ ] **Step 3: Run the test and verify it fails before implementation**

Run: `pytest tests/test_pages.py -v`

Expected: FAIL with `ModuleNotFoundError: No module named 'app'` or an import error for `create_app`.

- [ ] **Step 4: Implement minimal FastAPI app and dark shell**

Create `app/__init__.py`:

```python
"""Recipe site application package."""
```

Create `app/main.py`:

```python
from fastapi import FastAPI
from fastapi.responses import RedirectResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from starlette.requests import Request

templates = Jinja2Templates(directory="app/templates")


def create_app() -> FastAPI:
    app = FastAPI(title="Recipe Site")
    app.mount("/static", StaticFiles(directory="app/static"), name="static")

    @app.get("/", include_in_schema=False)
    def home() -> RedirectResponse:
        return RedirectResponse("/recipes", status_code=303)

    @app.get("/recipes", include_in_schema=False)
    def recipes_index(request: Request):
        return templates.TemplateResponse(
            "base.html",
            {
                "request": request,
                "title": "Recipes",
                "content_title": "Recipes",
                "empty_message": "No recipes yet",
            },
        )

    return app


app = create_app()
```

Create `app/templates/base.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{{ title or "Recipe Site" }}</title>
    <link rel="stylesheet" href="{{ url_for('static', path='/styles.css') }}">
  </head>
  <body>
    <header class="topbar">
      <a class="brand" href="/recipes">Recipe Site</a>
      <nav>
        <a href="/recipes">Recipes</a>
        <a href="/ingredients">Ingredients</a>
        <a href="/stock">Stock</a>
        <a href="/cart">Cart</a>
        <a href="/to-buy">To Buy</a>
      </nav>
    </header>
    <main class="page">
      <h1>{{ content_title }}</h1>
      {% if empty_message %}
        <p class="empty">{{ empty_message }}</p>
      {% endif %}
      {% block content %}{% endblock %}
    </main>
  </body>
</html>
```

Create `app/static/styles.css`:

```css
:root {
  color-scheme: dark;
  --bg: #101216;
  --panel: #181c22;
  --panel-2: #202632;
  --text: #f4f7fb;
  --muted: #a9b3c1;
  --line: #354052;
  --accent: #7dd3fc;
  --danger: #fb7185;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  background: var(--bg);
  color: var(--text);
  font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

a {
  color: var(--accent);
  text-decoration: none;
}

.topbar {
  align-items: center;
  background: #0b0d11;
  border-bottom: 1px solid var(--line);
  display: flex;
  gap: 24px;
  min-height: 56px;
  padding: 0 24px;
}

.brand {
  color: var(--text);
  font-weight: 800;
}

nav {
  display: flex;
  flex-wrap: wrap;
  gap: 14px;
}

.page {
  margin: 0 auto;
  max-width: 1180px;
  padding: 28px 20px 56px;
}

h1 {
  font-size: 30px;
  line-height: 1.2;
  margin: 0 0 20px;
}

.empty {
  background: var(--panel);
  border: 1px solid var(--line);
  border-radius: 8px;
  color: var(--muted);
  padding: 18px;
}
```

- [ ] **Step 5: Run the test and verify it passes**

Run: `pytest tests/test_pages.py -v`

Expected: 2 passed.

- [ ] **Step 6: Commit**

```bash
git add pyproject.toml app tests/test_pages.py
git commit -m "feat: scaffold FastAPI recipe app"
```

---

### Task 2: Database Models And Test Fixtures

**Files:**
- Create: `app/db.py`
- Create: `app/models.py`
- Create: `tests/conftest.py`
- Modify: `tests/test_pages.py`

- [ ] **Step 1: Write failing model fixture test**

Append to `tests/test_pages.py`:

```python
from sqlalchemy import select

from app.models import Ingredient


def test_database_fixture_can_persist_ingredient(db_session):
    db_session.add(
        Ingredient(
            name="Flour",
            stock_unit="kg",
            shopping_unit="kg",
        )
    )
    db_session.commit()

    ingredient = db_session.scalars(select(Ingredient)).one()

    assert ingredient.name == "Flour"
    assert ingredient.stock_unit == "kg"
    assert ingredient.shopping_unit == "kg"
```

- [ ] **Step 2: Run the test and verify it fails**

Run: `pytest tests/test_pages.py::test_database_fixture_can_persist_ingredient -v`

Expected: FAIL because `app.models` or `db_session` does not exist.

- [ ] **Step 3: Implement SQLAlchemy setup and models**

Create `app/db.py`:

```python
import os
from collections.abc import Generator

from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, Session, sessionmaker

DATABASE_URL = os.environ.get("DATABASE_URL", "sqlite:///data/recipes.db")


class Base(DeclarativeBase):
    pass


engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False} if DATABASE_URL.startswith("sqlite") else {},
)
SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)


def init_db() -> None:
    url = database_url()
    if url.startswith("sqlite:///") and not url.startswith("sqlite:///:memory:"):
        db_path = Path(url.removeprefix("sqlite:///"))
        if db_path.parent != Path("."):
            db_path.parent.mkdir(parents=True, exist_ok=True)
    Base.metadata.create_all(bind=engine)


def get_session() -> Generator[Session, None, None]:
    with SessionLocal() as session:
        yield session
```

Create `app/models.py`:

```python
from sqlalchemy import CheckConstraint, ForeignKey, String, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.db import Base


class Ingredient(Base):
    __tablename__ = "ingredients"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(160), unique=True, index=True)
    stock_unit: Mapped[str] = mapped_column(String(32), default="g")
    shopping_unit: Mapped[str] = mapped_column(String(32), default="g")

    conversions: Mapped[list["IngredientConversion"]] = relationship(
        back_populates="ingredient",
        cascade="all, delete-orphan",
    )


class IngredientConversion(Base):
    __tablename__ = "ingredient_conversions"
    __table_args__ = (CheckConstraint("grams_per_unit > 0", name="ck_conversion_positive"),)

    id: Mapped[int] = mapped_column(primary_key=True)
    ingredient_id: Mapped[int] = mapped_column(ForeignKey("ingredients.id", ondelete="CASCADE"))
    unit: Mapped[str] = mapped_column(String(32))
    grams_per_unit: Mapped[float]

    ingredient: Mapped[Ingredient] = relationship(back_populates="conversions")


class Recipe(Base):
    __tablename__ = "recipes"
    __table_args__ = (CheckConstraint("base_yield_quantity > 0", name="ck_recipe_yield_positive"),)

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(180), unique=True, index=True)
    base_yield_quantity: Mapped[float]
    base_yield_unit: Mapped[str] = mapped_column(String(32))
    instructions: Mapped[str] = mapped_column(Text, default="")

    items: Mapped[list["RecipeItem"]] = relationship(
        back_populates="recipe",
        cascade="all, delete-orphan",
        foreign_keys="RecipeItem.recipe_id",
        order_by="RecipeItem.sort_order",
    )


class RecipeItem(Base):
    __tablename__ = "recipe_items"
    __table_args__ = (
        CheckConstraint("quantity > 0", name="ck_recipe_item_quantity_positive"),
        CheckConstraint(
            "(ingredient_id IS NOT NULL AND subrecipe_id IS NULL) OR "
            "(ingredient_id IS NULL AND subrecipe_id IS NOT NULL)",
            name="ck_recipe_item_one_target",
        ),
    )

    id: Mapped[int] = mapped_column(primary_key=True)
    recipe_id: Mapped[int] = mapped_column(ForeignKey("recipes.id", ondelete="CASCADE"))
    sort_order: Mapped[int] = mapped_column(default=0)
    quantity: Mapped[float]
    unit: Mapped[str] = mapped_column(String(32))
    display_unit: Mapped[str] = mapped_column(String(32), default="g")
    note: Mapped[str] = mapped_column(String(240), default="")
    ingredient_id: Mapped[int | None] = mapped_column(ForeignKey("ingredients.id"))
    subrecipe_id: Mapped[int | None] = mapped_column(ForeignKey("recipes.id"))

    recipe: Mapped[Recipe] = relationship(
        back_populates="items",
        foreign_keys=[recipe_id],
    )
    ingredient: Mapped[Ingredient | None] = relationship()
    subrecipe: Mapped[Recipe | None] = relationship(foreign_keys=[subrecipe_id])


class StockItem(Base):
    __tablename__ = "stock_items"
    __table_args__ = (CheckConstraint("quantity > 0", name="ck_stock_quantity_positive"),)

    id: Mapped[int] = mapped_column(primary_key=True)
    ingredient_id: Mapped[int] = mapped_column(ForeignKey("ingredients.id"))
    quantity: Mapped[float]
    unit: Mapped[str] = mapped_column(String(32))
    location: Mapped[str] = mapped_column(String(120), default="")
    note: Mapped[str] = mapped_column(String(240), default="")

    ingredient: Mapped[Ingredient] = relationship()


class PlannedRecipe(Base):
    __tablename__ = "planned_recipes"
    __table_args__ = (CheckConstraint("target_quantity > 0", name="ck_planned_quantity_positive"),)

    id: Mapped[int] = mapped_column(primary_key=True)
    recipe_id: Mapped[int] = mapped_column(ForeignKey("recipes.id", ondelete="CASCADE"))
    target_quantity: Mapped[float]
    target_unit: Mapped[str] = mapped_column(String(32))
    note: Mapped[str] = mapped_column(String(240), default="")

    recipe: Mapped[Recipe] = relationship()
```

Create `tests/conftest.py`:

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

from app.db import Base


@pytest.fixture
def db_session() -> Session:
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    TestingSession = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)
    with TestingSession() as session:
        yield session
```

- [ ] **Step 4: Run model fixture test**

Run: `pytest tests/test_pages.py::test_database_fixture_can_persist_ingredient -v`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/db.py app/models.py tests/conftest.py tests/test_pages.py
git commit -m "feat: add recipe database models"
```

---

### Task 3: Unit Conversion Service

**Files:**
- Create: `app/services/__init__.py`
- Create: `app/services/units.py`
- Create: `tests/test_units.py`

- [ ] **Step 1: Write failing unit conversion tests**

Create `tests/test_units.py`:

```python
import pytest

from app.models import Ingredient, IngredientConversion
from app.services.units import ConversionError, convert_to_grams, format_amount


def test_global_weight_units_convert_to_grams(db_session):
    flour = Ingredient(name="Flour", stock_unit="kg", shopping_unit="kg")
    db_session.add(flour)
    db_session.commit()

    assert convert_to_grams(db_session, flour, 2, "kg") == pytest.approx(2000)
    assert convert_to_grams(db_session, flour, 8, "oz") == pytest.approx(226.796, rel=0.001)


def test_ingredient_specific_volume_conversion(db_session):
    oil = Ingredient(name="Olive oil", stock_unit="ml", shopping_unit="ml")
    db_session.add(oil)
    db_session.flush()
    db_session.add(IngredientConversion(ingredient_id=oil.id, unit="tbsp", grams_per_unit=13.5))
    db_session.commit()

    assert convert_to_grams(db_session, oil, 2, "tbsp") == pytest.approx(27)


def test_missing_conversion_raises(db_session):
    basil = Ingredient(name="Basil", stock_unit="bunch", shopping_unit="bunch")
    db_session.add(basil)
    db_session.commit()

    with pytest.raises(ConversionError, match="No conversion for Basil cup"):
        convert_to_grams(db_session, basil, 2, "cup")


def test_format_amount_uses_ingredient_specific_display_unit(db_session):
    coconut_milk = Ingredient(name="Coconut milk", stock_unit="can", shopping_unit="can")
    db_session.add(coconut_milk)
    db_session.flush()
    db_session.add(IngredientConversion(ingredient_id=coconut_milk.id, unit="can", grams_per_unit=382))
    db_session.commit()

    assert format_amount(db_session, coconut_milk, 764, "can") == "2 can"
```

- [ ] **Step 2: Run tests and verify they fail**

Run: `pytest tests/test_units.py -v`

Expected: FAIL because `app.services.units` does not exist.

- [ ] **Step 3: Implement unit conversion service**

Create `app/services/__init__.py`:

```python
"""Service-layer modules for recipe math and planning."""
```

Create `app/services/units.py`:

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

from app.models import Ingredient, IngredientConversion

GLOBAL_GRAMS_PER_UNIT = {
    "g": 1.0,
    "kg": 1000.0,
    "oz": 28.349523125,
    "lb": 453.59237,
}

CONTROLLED_UNITS = {
    "g",
    "kg",
    "oz",
    "lb",
    "ml",
    "l",
    "tsp",
    "tbsp",
    "cup",
    "each",
    "clove",
    "slice",
    "can",
    "carton",
    "bunch",
    "bag",
    "serving",
    "batch",
}


class ConversionError(ValueError):
    pass


def validate_unit(unit: str) -> None:
    if unit not in CONTROLLED_UNITS:
        raise ConversionError(f"Unsupported unit: {unit}")


def grams_per_unit(session: Session, ingredient: Ingredient, unit: str) -> float:
    validate_unit(unit)
    if unit in GLOBAL_GRAMS_PER_UNIT:
        return GLOBAL_GRAMS_PER_UNIT[unit]

    conversion = session.scalars(
        select(IngredientConversion).where(
            IngredientConversion.ingredient_id == ingredient.id,
            IngredientConversion.unit == unit,
        )
    ).first()
    if conversion is None:
        raise ConversionError(f"No conversion for {ingredient.name} {unit}")
    return conversion.grams_per_unit


def convert_to_grams(session: Session, ingredient: Ingredient, quantity: float, unit: str) -> float:
    if quantity <= 0:
        raise ConversionError("Quantity must be positive")
    return quantity * grams_per_unit(session, ingredient, unit)


def format_amount(session: Session, ingredient: Ingredient, grams: float, unit: str) -> str:
    per_unit = grams_per_unit(session, ingredient, unit)
    amount = grams / per_unit
    rounded = round(amount, 2)
    if rounded == int(rounded):
        rounded = int(rounded)
    return f"{rounded} {unit}"
```

- [ ] **Step 4: Run unit tests**

Run: `pytest tests/test_units.py -v`

Expected: 4 passed.

- [ ] **Step 5: Commit**

```bash
git add app/services tests/test_units.py
git commit -m "feat: add ingredient unit conversions"
```

---

### Task 4: Recipe Expansion And Scaling

**Files:**
- Create: `app/schemas.py`
- Create: `app/services/recipes.py`
- Create: `tests/test_recipes.py`

- [ ] **Step 1: Write failing recipe math tests**

Create `tests/test_recipes.py`:

```python
import pytest

from app.models import Ingredient, IngredientConversion, Recipe, RecipeItem
from app.services.recipes import RecipeCycleError, expand_recipe, scale_for_item


def seed_pesto_pasta(db_session):
    pasta = Ingredient(name="Pasta", stock_unit="lb", shopping_unit="lb")
    basil = Ingredient(name="Basil", stock_unit="bunch", shopping_unit="bunch")
    oil = Ingredient(name="Olive oil", stock_unit="ml", shopping_unit="ml")
    parmesan = Ingredient(name="Parmesan", stock_unit="g", shopping_unit="g")
    db_session.add_all([pasta, basil, oil, parmesan])
    db_session.flush()
    db_session.add_all(
        [
            IngredientConversion(ingredient_id=basil.id, unit="cup", grams_per_unit=20),
            IngredientConversion(ingredient_id=basil.id, unit="bunch", grams_per_unit=60),
            IngredientConversion(ingredient_id=oil.id, unit="cup", grams_per_unit=216),
        ]
    )
    pesto = Recipe(name="Pesto", base_yield_quantity=2, base_yield_unit="cup", instructions="Blend.")
    pasta_recipe = Recipe(name="Pesto Pasta", base_yield_quantity=4, base_yield_unit="serving", instructions="Toss.")
    db_session.add_all([pesto, pasta_recipe])
    db_session.flush()
    db_session.add_all(
        [
            RecipeItem(recipe_id=pesto.id, sort_order=1, ingredient_id=basil.id, quantity=4, unit="cup", display_unit="cup"),
            RecipeItem(recipe_id=pesto.id, sort_order=2, ingredient_id=oil.id, quantity=0.5, unit="cup", display_unit="cup"),
            RecipeItem(recipe_id=pasta_recipe.id, sort_order=1, ingredient_id=pasta.id, quantity=2, unit="lb", display_unit="lb"),
            RecipeItem(recipe_id=pasta_recipe.id, sort_order=2, subrecipe_id=pesto.id, quantity=1.5, unit="cup", display_unit="cup"),
            RecipeItem(recipe_id=pasta_recipe.id, sort_order=3, ingredient_id=parmesan.id, quantity=60, unit="g", display_unit="g"),
        ]
    )
    db_session.commit()
    return pasta_recipe, pesto, basil, oil, pasta, parmesan


def test_nested_recipe_expansion_combines_totals(db_session):
    pasta_recipe, _, basil, oil, pasta, parmesan = seed_pesto_pasta(db_session)

    result = expand_recipe(db_session, pasta_recipe, target_quantity=4, target_unit="serving")

    assert result.totals_by_ingredient[basil.id].grams == pytest.approx(60)
    assert result.totals_by_ingredient[oil.id].grams == pytest.approx(162)
    assert result.totals_by_ingredient[pasta.id].grams == pytest.approx(907.184, rel=0.001)
    assert result.totals_by_ingredient[parmesan.id].grams == pytest.approx(60)


def test_scaling_by_recipe_yield(db_session):
    pasta_recipe, _, basil, _, _, _ = seed_pesto_pasta(db_session)

    result = expand_recipe(db_session, pasta_recipe, target_quantity=8, target_unit="serving")

    assert result.scale == pytest.approx(2)
    assert result.totals_by_ingredient[basil.id].grams == pytest.approx(120)


def test_scaling_by_direct_item_quantity(db_session):
    pasta_recipe, _, _, _, pasta, _ = seed_pesto_pasta(db_session)
    pasta_item = next(item for item in pasta_recipe.items if item.ingredient_id == pasta.id)

    scale = scale_for_item(db_session, pasta_item, target_quantity=3, target_unit="lb")

    assert scale == pytest.approx(1.5)


def test_cycle_detection_rejects_recursive_recipes(db_session):
    a = Recipe(name="A", base_yield_quantity=1, base_yield_unit="batch", instructions="")
    b = Recipe(name="B", base_yield_quantity=1, base_yield_unit="batch", instructions="")
    db_session.add_all([a, b])
    db_session.flush()
    db_session.add_all(
        [
            RecipeItem(recipe_id=a.id, sort_order=1, subrecipe_id=b.id, quantity=1, unit="batch", display_unit="batch"),
            RecipeItem(recipe_id=b.id, sort_order=1, subrecipe_id=a.id, quantity=1, unit="batch", display_unit="batch"),
        ]
    )
    db_session.commit()

    with pytest.raises(RecipeCycleError):
        expand_recipe(db_session, a, target_quantity=1, target_unit="batch")
```

- [ ] **Step 2: Run tests and verify they fail**

Run: `pytest tests/test_recipes.py -v`

Expected: FAIL because `app.services.recipes` does not exist.

- [ ] **Step 3: Implement recipe schemas and expansion**

Create `app/schemas.py`:

```python
from dataclasses import dataclass, field

from app.models import Ingredient, Recipe


@dataclass
class IngredientTotal:
    ingredient: Ingredient
    grams: float = 0.0


@dataclass
class ExpandedRecipe:
    recipe: Recipe
    scale: float
    totals_by_ingredient: dict[int, IngredientTotal] = field(default_factory=dict)
```

Create `app/services/recipes.py`:

```python
from sqlalchemy.orm import Session

from app.models import Recipe, RecipeItem
from app.schemas import ExpandedRecipe, IngredientTotal
from app.services.units import ConversionError, convert_to_grams


class RecipeCycleError(ValueError):
    pass


def _yield_scale(recipe: Recipe, target_quantity: float, target_unit: str) -> float:
    if target_quantity <= 0:
        raise ConversionError("Target quantity must be positive")
    if target_unit != recipe.base_yield_unit:
        raise ConversionError(f"Cannot scale {recipe.name} from {recipe.base_yield_unit} to {target_unit}")
    return target_quantity / recipe.base_yield_quantity


def scale_for_item(session: Session, item: RecipeItem, target_quantity: float, target_unit: str) -> float:
    if target_quantity <= 0:
        raise ConversionError("Target quantity must be positive")
    if item.ingredient is not None:
        base_grams = convert_to_grams(session, item.ingredient, item.quantity, item.unit)
        target_grams = convert_to_grams(session, item.ingredient, target_quantity, target_unit)
        return target_grams / base_grams
    if item.subrecipe is not None:
        if target_unit != item.unit:
            raise ConversionError(f"Cannot scale sub-recipe item from {item.unit} to {target_unit}")
        return target_quantity / item.quantity
    raise ConversionError("Recipe item has no ingredient or sub-recipe")


def expand_recipe(
    session: Session,
    recipe: Recipe,
    target_quantity: float,
    target_unit: str,
    _seen: set[int] | None = None,
) -> ExpandedRecipe:
    seen = set() if _seen is None else set(_seen)
    if recipe.id in seen:
        raise RecipeCycleError(f"Recipe cycle detected at {recipe.name}")
    seen.add(recipe.id)

    scale = _yield_scale(recipe, target_quantity, target_unit)
    expanded = ExpandedRecipe(recipe=recipe, scale=scale)

    for item in recipe.items:
        if item.ingredient is not None:
            grams = convert_to_grams(session, item.ingredient, item.quantity * scale, item.unit)
            current = expanded.totals_by_ingredient.setdefault(
                item.ingredient.id,
                IngredientTotal(ingredient=item.ingredient),
            )
            current.grams += grams
            continue

        if item.subrecipe is not None:
            sub_target = item.quantity * scale
            sub_expanded = expand_recipe(session, item.subrecipe, sub_target, item.unit, seen)
            for ingredient_id, total in sub_expanded.totals_by_ingredient.items():
                current = expanded.totals_by_ingredient.setdefault(
                    ingredient_id,
                    IngredientTotal(ingredient=total.ingredient),
                )
                current.grams += total.grams
            continue

        raise ConversionError(f"Recipe item {item.id} has no ingredient or sub-recipe")

    return expanded
```

- [ ] **Step 4: Run recipe tests**

Run: `pytest tests/test_recipes.py -v`

Expected: 4 passed.

- [ ] **Step 5: Commit**

```bash
git add app/schemas.py app/services/recipes.py tests/test_recipes.py
git commit -m "feat: expand nested recipes"
```

---

### Task 5: Planning And Shopping Calculations

**Files:**
- Create: `app/services/planning.py`
- Create: `tests/test_planning.py`

- [ ] **Step 1: Write failing planning tests**

Create `tests/test_planning.py`:

```python
import pytest

from app.models import Ingredient, PlannedRecipe, Recipe, RecipeItem, StockItem
from app.services.planning import compute_shopping_list


def test_shopping_list_subtracts_stock_and_formats_shortage(db_session):
    flour = Ingredient(name="Flour", stock_unit="kg", shopping_unit="kg")
    salt = Ingredient(name="Salt", stock_unit="g", shopping_unit="g")
    dough = Recipe(name="Dough", base_yield_quantity=2, base_yield_unit="serving", instructions="")
    db_session.add_all([flour, salt, dough])
    db_session.flush()
    db_session.add_all(
        [
            RecipeItem(recipe_id=dough.id, sort_order=1, ingredient_id=flour.id, quantity=1000, unit="g", display_unit="g"),
            RecipeItem(recipe_id=dough.id, sort_order=2, ingredient_id=salt.id, quantity=20, unit="g", display_unit="g"),
            StockItem(ingredient_id=flour.id, quantity=0.25, unit="kg", location="pantry"),
            PlannedRecipe(recipe_id=dough.id, target_quantity=4, target_unit="serving"),
        ]
    )
    db_session.commit()

    shopping = compute_shopping_list(db_session)

    assert shopping[flour.id].needed_grams == pytest.approx(2000)
    assert shopping[flour.id].stock_grams == pytest.approx(250)
    assert shopping[flour.id].shortage_grams == pytest.approx(1750)
    assert shopping[flour.id].display == "1.75 kg"
    assert shopping[salt.id].display == "40 g"
```

- [ ] **Step 2: Run tests and verify they fail**

Run: `pytest tests/test_planning.py -v`

Expected: FAIL because `app.services.planning` does not exist.

- [ ] **Step 3: Implement planning service**

Create `app/services/planning.py`:

```python
from dataclasses import dataclass

from sqlalchemy import select
from sqlalchemy.orm import Session

from app.models import Ingredient, PlannedRecipe, StockItem
from app.services.recipes import expand_recipe
from app.services.units import convert_to_grams, format_amount


@dataclass
class ShoppingLine:
    ingredient: Ingredient
    needed_grams: float
    stock_grams: float
    shortage_grams: float
    display: str


def compute_stock_totals(session: Session) -> dict[int, float]:
    totals: dict[int, float] = {}
    for stock in session.scalars(select(StockItem)).all():
        grams = convert_to_grams(session, stock.ingredient, stock.quantity, stock.unit)
        totals[stock.ingredient_id] = totals.get(stock.ingredient_id, 0.0) + grams
    return totals


def compute_needed_totals(session: Session) -> dict[int, float]:
    totals: dict[int, float] = {}
    for planned in session.scalars(select(PlannedRecipe)).all():
        expanded = expand_recipe(session, planned.recipe, planned.target_quantity, planned.target_unit)
        for ingredient_id, total in expanded.totals_by_ingredient.items():
            totals[ingredient_id] = totals.get(ingredient_id, 0.0) + total.grams
    return totals


def compute_shopping_list(session: Session) -> dict[int, ShoppingLine]:
    needed = compute_needed_totals(session)
    stock = compute_stock_totals(session)
    lines: dict[int, ShoppingLine] = {}

    for ingredient_id, needed_grams in needed.items():
        stock_grams = stock.get(ingredient_id, 0.0)
        shortage = max(needed_grams - stock_grams, 0.0)
        if shortage <= 0:
            continue
        ingredient = session.get(Ingredient, ingredient_id)
        if ingredient is None:
            continue
        lines[ingredient_id] = ShoppingLine(
            ingredient=ingredient,
            needed_grams=needed_grams,
            stock_grams=stock_grams,
            shortage_grams=shortage,
            display=format_amount(session, ingredient, shortage, ingredient.shopping_unit),
        )

    return lines
```

- [ ] **Step 4: Run planning tests**

Run: `pytest tests/test_planning.py -v`

Expected: 1 passed.

- [ ] **Step 5: Commit**

```bash
git add app/services/planning.py tests/test_planning.py
git commit -m "feat: compute shopping shortages"
```

---

### Task 6: Application Database Wiring

**Files:**
- Modify: `app/main.py`
- Modify: `tests/test_pages.py`

- [ ] **Step 1: Write failing startup/database smoke test**

Append to `tests/test_pages.py`:

```python
def test_create_app_initializes_database(tmp_path, monkeypatch):
    db_file = tmp_path / "recipes.db"
    monkeypatch.setenv("DATABASE_URL", f"sqlite:///{db_file}")

    app = create_app()

    assert app.title == "Recipe Site"
    assert db_file.exists()
```

- [ ] **Step 2: Run the test and verify it fails**

Run: `pytest tests/test_pages.py::test_create_app_initializes_database -v`

Expected: FAIL because `create_app` does not initialize the database after reading the patched env var.

- [ ] **Step 3: Refactor database setup for runtime initialization**

Modify `app/db.py` so engine creation can be refreshed:

```python
import os
from collections.abc import Generator
from pathlib import Path

from sqlalchemy import Engine, create_engine
from sqlalchemy.orm import DeclarativeBase, Session, sessionmaker


class Base(DeclarativeBase):
    pass


def database_url() -> str:
    return os.environ.get("DATABASE_URL", "sqlite:///data/recipes.db")


def make_engine(url: str | None = None) -> Engine:
    selected_url = url or database_url()
    return create_engine(
        selected_url,
        connect_args={"check_same_thread": False} if selected_url.startswith("sqlite") else {},
    )


engine = make_engine()
SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)


def configure_database(url: str | None = None) -> None:
    global engine, SessionLocal
    engine = make_engine(url)
    SessionLocal.configure(bind=engine)


def init_db() -> None:
    url = database_url()
    if url.startswith("sqlite:///") and not url.startswith("sqlite:///:memory:"):
        db_path = Path(url.removeprefix("sqlite:///"))
        if db_path.parent != Path("."):
            db_path.parent.mkdir(parents=True, exist_ok=True)
    Base.metadata.create_all(bind=engine)


def get_session() -> Generator[Session, None, None]:
    with SessionLocal() as session:
        yield session
```

Modify `app/main.py` to call database configuration/init:

```python
from fastapi import FastAPI
from fastapi.responses import RedirectResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from starlette.requests import Request

from app.db import configure_database, init_db

templates = Jinja2Templates(directory="app/templates")


def create_app() -> FastAPI:
    configure_database()
    init_db()
    app = FastAPI(title="Recipe Site")
    app.mount("/static", StaticFiles(directory="app/static"), name="static")

    @app.get("/", include_in_schema=False)
    def home() -> RedirectResponse:
        return RedirectResponse("/recipes", status_code=303)

    @app.get("/recipes", include_in_schema=False)
    def recipes_index(request: Request):
        return templates.TemplateResponse(
            "base.html",
            {
                "request": request,
                "title": "Recipes",
                "content_title": "Recipes",
                "empty_message": "No recipes yet",
            },
        )

    return app


app = create_app()
```

- [ ] **Step 4: Run page tests**

Run: `pytest tests/test_pages.py -v`

Expected: all tests in `tests/test_pages.py` pass.

- [ ] **Step 5: Commit**

```bash
git add app/db.py app/main.py tests/test_pages.py
git commit -m "feat: initialize app database"
```

---

### Task 7: Ingredients And Conversion Web Forms

**Files:**
- Create: `app/routes/ingredients.py`
- Create: `app/templates/ingredients/index.html`
- Modify: `app/main.py`
- Modify: `tests/test_pages.py`

- [ ] **Step 1: Write failing ingredient page/form tests**

Append to `tests/test_pages.py`:

```python
def test_ingredients_page_creates_ingredient(tmp_path, monkeypatch):
    db_file = tmp_path / "recipes.db"
    monkeypatch.setenv("DATABASE_URL", f"sqlite:///{db_file}")
    client = TestClient(create_app())

    response = client.post(
        "/ingredients",
        data={"name": "Coconut milk", "stock_unit": "can", "shopping_unit": "can"},
        follow_redirects=False,
    )

    assert response.status_code == 303
    assert response.headers["location"] == "/ingredients"
    page = client.get("/ingredients")
    assert "Coconut milk" in page.text


def test_ingredient_conversion_can_be_added(tmp_path, monkeypatch):
    db_file = tmp_path / "recipes.db"
    monkeypatch.setenv("DATABASE_URL", f"sqlite:///{db_file}")
    client = TestClient(create_app())
    client.post("/ingredients", data={"name": "Coconut milk", "stock_unit": "can", "shopping_unit": "can"})

    response = client.post(
        "/ingredients/1/conversions",
        data={"unit": "can", "grams_per_unit": "382"},
        follow_redirects=False,
    )

    assert response.status_code == 303
    page = client.get("/ingredients")
    assert "can = 382 g" in page.text
```

- [ ] **Step 2: Run tests and verify they fail**

Run: `pytest tests/test_pages.py::test_ingredients_page_creates_ingredient tests/test_pages.py::test_ingredient_conversion_can_be_added -v`

Expected: FAIL with 404 for `/ingredients`.

- [ ] **Step 3: Implement ingredients routes and template**

Create `app/routes/ingredients.py`:

```python
from fastapi import APIRouter, Depends, Form
from fastapi.responses import RedirectResponse
from sqlalchemy import select
from sqlalchemy.orm import Session
from starlette.requests import Request

from app.db import get_session
from app.main import templates
from app.models import Ingredient, IngredientConversion

router = APIRouter()


@router.get("/ingredients", include_in_schema=False)
def index(request: Request, session: Session = Depends(get_session)):
    ingredients = session.scalars(select(Ingredient).order_by(Ingredient.name)).all()
    return templates.TemplateResponse(
        "ingredients/index.html",
        {"request": request, "title": "Ingredients", "content_title": "Ingredients", "ingredients": ingredients},
    )


@router.post("/ingredients", include_in_schema=False)
def create_ingredient(
    name: str = Form(...),
    stock_unit: str = Form("g"),
    shopping_unit: str = Form("g"),
    session: Session = Depends(get_session),
):
    session.add(Ingredient(name=name.strip(), stock_unit=stock_unit, shopping_unit=shopping_unit))
    session.commit()
    return RedirectResponse("/ingredients", status_code=303)


@router.post("/ingredients/{ingredient_id}/conversions", include_in_schema=False)
def create_conversion(
    ingredient_id: int,
    unit: str = Form(...),
    grams_per_unit: float = Form(...),
    session: Session = Depends(get_session),
):
    session.add(IngredientConversion(ingredient_id=ingredient_id, unit=unit, grams_per_unit=grams_per_unit))
    session.commit()
    return RedirectResponse("/ingredients", status_code=303)
```

Create `app/templates/ingredients/index.html`:

```html
{% extends "base.html" %}

{% block content %}
  <section class="panel">
    <h2>Add Ingredient</h2>
    <form method="post" action="/ingredients" class="grid-form">
      <label>Name <input name="name" required></label>
      <label>Stock unit <input name="stock_unit" value="g" required></label>
      <label>Shopping unit <input name="shopping_unit" value="g" required></label>
      <button type="submit">Add</button>
    </form>
  </section>

  <section class="panel">
    <h2>Ingredients</h2>
    {% if not ingredients %}
      <p class="empty">No ingredients yet</p>
    {% endif %}
    {% for ingredient in ingredients %}
      <article class="row-card">
        <div>
          <strong>{{ ingredient.name }}</strong>
          <p>Stock: {{ ingredient.stock_unit }} · Shopping: {{ ingredient.shopping_unit }}</p>
          {% for conversion in ingredient.conversions %}
            <p>{{ conversion.unit }} = {{ conversion.grams_per_unit|round(2) }} g</p>
          {% endfor %}
        </div>
        <form method="post" action="/ingredients/{{ ingredient.id }}/conversions" class="inline-form">
          <input name="unit" placeholder="unit" required>
          <input name="grams_per_unit" type="number" step="0.01" min="0.01" placeholder="grams" required>
          <button type="submit">Add conversion</button>
        </form>
      </article>
    {% endfor %}
  </section>
{% endblock %}
```

Modify `app/main.py` inside `create_app()` after mounting static files:

```python
    from app.routes.ingredients import router as ingredients_router

    app.include_router(ingredients_router)
```

- [ ] **Step 4: Add form CSS**

Append to `app/static/styles.css`:

```css
.panel {
  background: var(--panel);
  border: 1px solid var(--line);
  border-radius: 8px;
  margin-bottom: 18px;
  padding: 18px;
}

.grid-form,
.inline-form {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
}

label {
  color: var(--muted);
  display: grid;
  gap: 6px;
}

input,
textarea,
select,
button {
  background: #0f141b;
  border: 1px solid var(--line);
  border-radius: 6px;
  color: var(--text);
  font: inherit;
  min-height: 38px;
  padding: 8px 10px;
}

button {
  background: var(--accent);
  color: #06202c;
  cursor: pointer;
  font-weight: 800;
}

.row-card {
  align-items: start;
  background: var(--panel-2);
  border: 1px solid var(--line);
  border-radius: 8px;
  display: grid;
  gap: 14px;
  grid-template-columns: minmax(0, 1fr) minmax(280px, 420px);
  margin-top: 12px;
  padding: 14px;
}

.row-card p {
  color: var(--muted);
  margin: 6px 0 0;
}

@media (max-width: 760px) {
  .row-card {
    grid-template-columns: 1fr;
  }
}
```

- [ ] **Step 5: Run ingredient page tests**

Run: `pytest tests/test_pages.py::test_ingredients_page_creates_ingredient tests/test_pages.py::test_ingredient_conversion_can_be_added -v`

Expected: 2 passed.

- [ ] **Step 6: Commit**

```bash
git add app/main.py app/routes/ingredients.py app/templates/ingredients/index.html app/static/styles.css tests/test_pages.py
git commit -m "feat: manage ingredients and conversions"
```

---

### Task 8: Recipe Web Pages And Forms

**Files:**
- Create: `app/routes/recipes.py`
- Create: `app/templates/recipes/index.html`
- Create: `app/templates/recipes/detail.html`
- Create: `app/templates/recipes/form.html`
- Modify: `app/main.py`
- Modify: `tests/test_pages.py`

- [ ] **Step 1: Write failing recipe form tests**

Append to `tests/test_pages.py`:

```python
def test_recipe_can_be_created_with_ingredient_row(tmp_path, monkeypatch):
    db_file = tmp_path / "recipes.db"
    monkeypatch.setenv("DATABASE_URL", f"sqlite:///{db_file}")
    client = TestClient(create_app())
    client.post("/ingredients", data={"name": "Flour", "stock_unit": "kg", "shopping_unit": "kg"})

    response = client.post(
        "/recipes",
        data={
            "name": "Flatbread",
            "base_yield_quantity": "4",
            "base_yield_unit": "serving",
            "instructions": "Mix and cook.",
            "item_kind": ["ingredient"],
            "ingredient_id": ["1"],
            "subrecipe_id": [""],
            "quantity": ["500"],
            "unit": ["g"],
            "display_unit": ["g"],
            "note": ["flour"],
        },
        follow_redirects=False,
    )

    assert response.status_code == 303
    page = client.get("/recipes/1")
    assert "Flatbread" in page.text
    assert "500 g" in page.text
    assert "Mix and cook." in page.text
```

- [ ] **Step 2: Run test and verify it fails**

Run: `pytest tests/test_pages.py::test_recipe_can_be_created_with_ingredient_row -v`

Expected: FAIL because recipe routes/forms are not implemented.

- [ ] **Step 3: Implement recipe routes**

Create `app/routes/recipes.py`:

```python
from fastapi import APIRouter, Depends, Form
from fastapi.responses import RedirectResponse
from sqlalchemy import select
from sqlalchemy.orm import Session
from starlette.requests import Request

from app.db import get_session
from app.main import templates
from app.models import Ingredient, Recipe, RecipeItem
from app.services.recipes import expand_recipe

router = APIRouter()


@router.get("/recipes", include_in_schema=False)
def index(request: Request, session: Session = Depends(get_session)):
    recipes = session.scalars(select(Recipe).order_by(Recipe.name)).all()
    return templates.TemplateResponse(
        "recipes/index.html",
        {"request": request, "title": "Recipes", "content_title": "Recipes", "recipes": recipes},
    )


@router.get("/recipes/new", include_in_schema=False)
def new(request: Request, session: Session = Depends(get_session)):
    return _form_response(request, session, Recipe(name="", base_yield_quantity=1, base_yield_unit="serving", instructions=""))


@router.post("/recipes", include_in_schema=False)
def create(
    name: str = Form(...),
    base_yield_quantity: float = Form(...),
    base_yield_unit: str = Form(...),
    instructions: str = Form(""),
    item_kind: list[str] = Form([]),
    ingredient_id: list[str] = Form([]),
    subrecipe_id: list[str] = Form([]),
    quantity: list[float] = Form([]),
    unit: list[str] = Form([]),
    display_unit: list[str] = Form([]),
    note: list[str] = Form([]),
    session: Session = Depends(get_session),
):
    recipe = Recipe(
        name=name.strip(),
        base_yield_quantity=base_yield_quantity,
        base_yield_unit=base_yield_unit,
        instructions=instructions,
    )
    session.add(recipe)
    session.flush()
    for index, kind in enumerate(item_kind):
        if not kind:
            continue
        session.add(
            RecipeItem(
                recipe_id=recipe.id,
                sort_order=index + 1,
                ingredient_id=int(ingredient_id[index]) if kind == "ingredient" and ingredient_id[index] else None,
                subrecipe_id=int(subrecipe_id[index]) if kind == "subrecipe" and subrecipe_id[index] else None,
                quantity=quantity[index],
                unit=unit[index],
                display_unit=display_unit[index],
                note=note[index],
            )
        )
    session.commit()
    return RedirectResponse(f"/recipes/{recipe.id}", status_code=303)


@router.get("/recipes/{recipe_id}", include_in_schema=False)
def detail(recipe_id: int, request: Request, session: Session = Depends(get_session)):
    recipe = session.get(Recipe, recipe_id)
    if recipe is None:
        return RedirectResponse("/recipes", status_code=303)
    expanded = expand_recipe(session, recipe, recipe.base_yield_quantity, recipe.base_yield_unit)
    return templates.TemplateResponse(
        "recipes/detail.html",
        {
            "request": request,
            "title": recipe.name,
            "content_title": recipe.name,
            "recipe": recipe,
            "expanded": expanded,
        },
    )


def _form_response(request: Request, session: Session, recipe: Recipe):
    ingredients = session.scalars(select(Ingredient).order_by(Ingredient.name)).all()
    recipes = session.scalars(select(Recipe).order_by(Recipe.name)).all()
    return templates.TemplateResponse(
        "recipes/form.html",
        {
            "request": request,
            "title": "New Recipe",
            "content_title": "New Recipe",
            "recipe": recipe,
            "ingredients": ingredients,
            "recipes": recipes,
        },
    )
```

Modify `app/main.py`:

```python
    from app.routes.ingredients import router as ingredients_router
    from app.routes.recipes import router as recipes_router

    app.include_router(ingredients_router)
    app.include_router(recipes_router)
```

Remove the temporary inline `/recipes` route from `app/main.py`.

- [ ] **Step 4: Create recipe templates**

Create `app/templates/recipes/index.html`:

```html
{% extends "base.html" %}

{% block content %}
  <p><a class="button-link" href="/recipes/new">New recipe</a></p>
  {% if not recipes %}
    <p class="empty">No recipes yet</p>
  {% endif %}
  <section class="card-grid">
    {% for recipe in recipes %}
      <a class="recipe-card" href="/recipes/{{ recipe.id }}">
        <strong>{{ recipe.name }}</strong>
        <span>{{ recipe.base_yield_quantity }} {{ recipe.base_yield_unit }}</span>
      </a>
    {% endfor %}
  </section>
{% endblock %}
```

Create `app/templates/recipes/form.html`:

```html
{% extends "base.html" %}

{% block content %}
  <form method="post" action="/recipes" class="panel stack">
    <label>Name <input name="name" value="{{ recipe.name }}" required></label>
    <label>Base yield quantity <input name="base_yield_quantity" type="number" step="0.01" min="0.01" value="{{ recipe.base_yield_quantity }}" required></label>
    <label>Base yield unit <input name="base_yield_unit" value="{{ recipe.base_yield_unit }}" required></label>
    <label>Instructions <textarea name="instructions" rows="8">{{ recipe.instructions }}</textarea></label>

    <h2>Items</h2>
    <div id="recipe-items">
      <div class="recipe-item-row">
        <select name="item_kind">
          <option value="ingredient">Ingredient</option>
          <option value="subrecipe">Sub-recipe</option>
        </select>
        <select name="ingredient_id">
          <option value="">Ingredient</option>
          {% for ingredient in ingredients %}
            <option value="{{ ingredient.id }}">{{ ingredient.name }}</option>
          {% endfor %}
        </select>
        <select name="subrecipe_id">
          <option value="">Sub-recipe</option>
          {% for option_recipe in recipes %}
            <option value="{{ option_recipe.id }}">{{ option_recipe.name }}</option>
          {% endfor %}
        </select>
        <input name="quantity" type="number" step="0.01" min="0.01" placeholder="Qty" required>
        <input name="unit" placeholder="Unit" required>
        <input name="display_unit" placeholder="Display unit" required>
        <input name="note" placeholder="Note">
      </div>
    </div>
    <button type="button" data-add-recipe-row>Add row</button>
    <button type="submit">Save recipe</button>
  </form>
  <script src="{{ url_for('static', path='/app.js') }}"></script>
{% endblock %}
```

Create `app/templates/recipes/detail.html`:

```html
{% extends "base.html" %}

{% block content %}
  <section class="panel">
    <p>Base yield: {{ recipe.base_yield_quantity }} {{ recipe.base_yield_unit }}</p>
    <pre class="instructions">{{ recipe.instructions }}</pre>
  </section>

  <section class="panel">
    <h2>Items</h2>
    {% for item in recipe.items %}
      <p>
        {{ item.quantity }} {{ item.display_unit }}
        {% if item.ingredient %}{{ item.ingredient.name }}{% endif %}
        {% if item.subrecipe %}{{ item.subrecipe.name }}{% endif %}
        {% if item.note %}<span class="muted">({{ item.note }})</span>{% endif %}
      </p>
    {% endfor %}
  </section>

  <section class="panel">
    <h2>Combined totals</h2>
    {% for total in expanded.totals_by_ingredient.values() %}
      <p>{{ total.ingredient.name }}: {{ total.grams|round(1) }} g</p>
    {% endfor %}
  </section>
{% endblock %}
```

- [ ] **Step 5: Add recipe JavaScript and CSS**

Create `app/static/app.js`:

```javascript
document.addEventListener("click", (event) => {
  const addButton = event.target.closest("[data-add-recipe-row]");
  if (!addButton) return;
  const container = document.querySelector("#recipe-items");
  const firstRow = container.querySelector(".recipe-item-row");
  const clone = firstRow.cloneNode(true);
  clone.querySelectorAll("input").forEach((input) => {
    input.value = "";
  });
  clone.querySelectorAll("select").forEach((select) => {
    select.selectedIndex = 0;
  });
  container.appendChild(clone);
});
```

Append to `app/static/styles.css`:

```css
.button-link {
  background: var(--accent);
  border-radius: 6px;
  color: #06202c;
  display: inline-block;
  font-weight: 800;
  padding: 9px 12px;
}

.card-grid {
  display: grid;
  gap: 12px;
  grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
}

.recipe-card {
  background: var(--panel);
  border: 1px solid var(--line);
  border-radius: 8px;
  color: var(--text);
  display: grid;
  gap: 8px;
  padding: 14px;
}

.recipe-card span,
.muted {
  color: var(--muted);
}

.stack {
  display: grid;
  gap: 14px;
}

.recipe-item-row {
  display: grid;
  gap: 8px;
  grid-template-columns: 130px 1fr 1fr 100px 90px 120px 1fr;
  margin-bottom: 10px;
}

.instructions {
  color: var(--text);
  font: inherit;
  white-space: pre-wrap;
}

@media (max-width: 1000px) {
  .recipe-item-row {
    grid-template-columns: 1fr 1fr;
  }
}
```

- [ ] **Step 6: Run recipe form test**

Run: `pytest tests/test_pages.py::test_recipe_can_be_created_with_ingredient_row -v`

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add app/main.py app/routes/recipes.py app/templates/recipes app/static app/static/app.js tests/test_pages.py
git commit -m "feat: create and view recipes"
```

---

### Task 9: Stock, Cart, And To Buy Pages

**Files:**
- Create: `app/routes/stock.py`
- Create: `app/routes/cart.py`
- Create: `app/routes/shopping.py`
- Create: `app/templates/stock/index.html`
- Create: `app/templates/cart/index.html`
- Create: `app/templates/shopping/index.html`
- Modify: `app/main.py`
- Modify: `tests/test_pages.py`

- [ ] **Step 1: Write failing end-to-end page flow test**

Append to `tests/test_pages.py`:

```python
def test_stock_cart_and_to_buy_flow(tmp_path, monkeypatch):
    db_file = tmp_path / "recipes.db"
    monkeypatch.setenv("DATABASE_URL", f"sqlite:///{db_file}")
    client = TestClient(create_app())
    client.post("/ingredients", data={"name": "Flour", "stock_unit": "kg", "shopping_unit": "kg"})
    client.post(
        "/recipes",
        data={
            "name": "Flatbread",
            "base_yield_quantity": "4",
            "base_yield_unit": "serving",
            "instructions": "Mix and cook.",
            "item_kind": ["ingredient"],
            "ingredient_id": ["1"],
            "subrecipe_id": [""],
            "quantity": ["1000"],
            "unit": ["g"],
            "display_unit": ["g"],
            "note": [""],
        },
    )

    assert client.post("/stock", data={"ingredient_id": "1", "quantity": "0.25", "unit": "kg", "location": "pantry", "note": ""}).status_code == 200
    assert client.post("/cart", data={"recipe_id": "1", "target_quantity": "4", "target_unit": "serving", "note": ""}).status_code == 200

    shopping = client.get("/to-buy")

    assert shopping.status_code == 200
    assert "Flour" in shopping.text
    assert "0.75 kg" in shopping.text
```

- [ ] **Step 2: Run test and verify it fails**

Run: `pytest tests/test_pages.py::test_stock_cart_and_to_buy_flow -v`

Expected: FAIL with 404 for `/stock`.

- [ ] **Step 3: Implement stock route/template**

Create `app/routes/stock.py`:

```python
from fastapi import APIRouter, Depends, Form
from sqlalchemy import select
from sqlalchemy.orm import Session
from starlette.requests import Request

from app.db import get_session
from app.main import templates
from app.models import Ingredient, StockItem

router = APIRouter()


@router.get("/stock", include_in_schema=False)
def index(request: Request, session: Session = Depends(get_session)):
    ingredients = session.scalars(select(Ingredient).order_by(Ingredient.name)).all()
    stock_items = session.scalars(select(StockItem).order_by(StockItem.id)).all()
    return templates.TemplateResponse(
        "stock/index.html",
        {"request": request, "title": "Stock", "content_title": "Stock", "ingredients": ingredients, "stock_items": stock_items},
    )


@router.post("/stock", include_in_schema=False)
def create(
    request: Request,
    ingredient_id: int = Form(...),
    quantity: float = Form(...),
    unit: str = Form(...),
    location: str = Form(""),
    note: str = Form(""),
    session: Session = Depends(get_session),
):
    session.add(StockItem(ingredient_id=ingredient_id, quantity=quantity, unit=unit, location=location, note=note))
    session.commit()
    return index(request, session)
```

Create `app/templates/stock/index.html`:

```html
{% extends "base.html" %}

{% block content %}
  <section class="panel">
    <h2>Add Stock</h2>
    <form method="post" action="/stock" class="grid-form">
      <select name="ingredient_id" required>
        {% for ingredient in ingredients %}
          <option value="{{ ingredient.id }}">{{ ingredient.name }}</option>
        {% endfor %}
      </select>
      <input name="quantity" type="number" step="0.01" min="0.01" placeholder="Qty" required>
      <input name="unit" placeholder="Unit" required>
      <input name="location" placeholder="Location">
      <input name="note" placeholder="Note">
      <button type="submit">Add</button>
    </form>
  </section>
  <section class="panel">
    <h2>Current Stock</h2>
    {% for item in stock_items %}
      <p>{{ item.ingredient.name }}: {{ item.quantity }} {{ item.unit }}{% if item.location %} · {{ item.location }}{% endif %}</p>
    {% else %}
      <p class="empty">No stock yet</p>
    {% endfor %}
  </section>
{% endblock %}
```

- [ ] **Step 4: Implement cart and shopping routes/templates**

Create `app/routes/cart.py`:

```python
from fastapi import APIRouter, Depends, Form
from sqlalchemy import select
from sqlalchemy.orm import Session
from starlette.requests import Request

from app.db import get_session
from app.main import templates
from app.models import PlannedRecipe, Recipe

router = APIRouter()


@router.get("/cart", include_in_schema=False)
def index(request: Request, session: Session = Depends(get_session)):
    recipes = session.scalars(select(Recipe).order_by(Recipe.name)).all()
    planned = session.scalars(select(PlannedRecipe).order_by(PlannedRecipe.id)).all()
    return templates.TemplateResponse(
        "cart/index.html",
        {"request": request, "title": "Cart", "content_title": "Cart", "recipes": recipes, "planned": planned},
    )


@router.post("/cart", include_in_schema=False)
def create(
    request: Request,
    recipe_id: int = Form(...),
    target_quantity: float = Form(...),
    target_unit: str = Form(...),
    note: str = Form(""),
    session: Session = Depends(get_session),
):
    session.add(PlannedRecipe(recipe_id=recipe_id, target_quantity=target_quantity, target_unit=target_unit, note=note))
    session.commit()
    return index(request, session)
```

Create `app/routes/shopping.py`:

```python
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from starlette.requests import Request

from app.db import get_session
from app.main import templates
from app.services.planning import compute_shopping_list

router = APIRouter()


@router.get("/to-buy", include_in_schema=False)
def index(request: Request, session: Session = Depends(get_session)):
    lines = compute_shopping_list(session)
    return templates.TemplateResponse(
        "shopping/index.html",
        {"request": request, "title": "To Buy", "content_title": "To Buy", "lines": lines.values()},
    )
```

Create `app/templates/cart/index.html`:

```html
{% extends "base.html" %}

{% block content %}
  <section class="panel">
    <h2>Add Planned Recipe</h2>
    <form method="post" action="/cart" class="grid-form">
      <select name="recipe_id" required>
        {% for recipe in recipes %}
          <option value="{{ recipe.id }}">{{ recipe.name }}</option>
        {% endfor %}
      </select>
      <input name="target_quantity" type="number" step="0.01" min="0.01" placeholder="Target qty" required>
      <input name="target_unit" placeholder="Target unit" required>
      <input name="note" placeholder="Note">
      <button type="submit">Add</button>
    </form>
  </section>
  <section class="panel">
    <h2>Planning To Cook</h2>
    {% for item in planned %}
      <p>{{ item.recipe.name }}: {{ item.target_quantity }} {{ item.target_unit }}</p>
    {% else %}
      <p class="empty">No planned recipes yet</p>
    {% endfor %}
  </section>
{% endblock %}
```

Create `app/templates/shopping/index.html`:

```html
{% extends "base.html" %}

{% block content %}
  <section class="panel">
    {% for line in lines %}
      <p><strong>{{ line.ingredient.name }}</strong>: {{ line.display }}</p>
    {% else %}
      <p class="empty">Nothing to buy</p>
    {% endfor %}
  </section>
{% endblock %}
```

Modify `app/main.py` to include these routers:

```python
    from app.routes.cart import router as cart_router
    from app.routes.shopping import router as shopping_router
    from app.routes.stock import router as stock_router

    app.include_router(stock_router)
    app.include_router(cart_router)
    app.include_router(shopping_router)
```

- [ ] **Step 5: Run flow test**

Run: `pytest tests/test_pages.py::test_stock_cart_and_to_buy_flow -v`

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add app/main.py app/routes/stock.py app/routes/cart.py app/routes/shopping.py app/templates/stock app/templates/cart app/templates/shopping tests/test_pages.py
git commit -m "feat: add stock cart and shopping pages"
```

---

### Task 10: Docker, README, And Full Verification

**Files:**
- Create: `Dockerfile`
- Create: `docker-compose.yml`
- Modify: `README.md`

- [ ] **Step 1: Write Docker files**

Create `Dockerfile`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY pyproject.toml ./
RUN pip install --no-cache-dir ".[dev]"

COPY app ./app

RUN mkdir -p /data
ENV DATABASE_URL=sqlite:////data/recipes.db

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Create `docker-compose.yml`:

```yaml
services:
  recipes:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: sqlite:////data/recipes.db
    volumes:
      - recipe-data:/data

volumes:
  recipe-data:
```

- [ ] **Step 2: Update README**

Replace `README.md` with:

```markdown
# recipessite

A single-user local recipe app for nested recipes, ingredient conversions, stock tracking, planned cooking, and computed shopping needs.

## Local Development

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
pytest
uvicorn app.main:app --reload
```

Open `http://localhost:8000`.

## Docker

```bash
docker compose up --build
```

The SQLite database is stored in the `recipe-data` Docker volume.

## Current Scope

- Recipes and reusable sub-recipes.
- Ingredient-specific unit conversions into grams.
- Stock tracking.
- Cart of planned recipes.
- To Buy list computed from planned recipe totals minus stock.
```

- [ ] **Step 3: Run the full test suite**

Run: `pytest -v`

Expected: all tests pass.

- [ ] **Step 4: Build Docker image**

Run: `docker compose build`

Expected: build completes successfully.

- [ ] **Step 5: Start app locally for manual check**

Run: `uvicorn app.main:app --host 127.0.0.1 --port 8000`

Expected: server starts and prints `Uvicorn running on http://127.0.0.1:8000`.

Open these pages manually:

- `http://127.0.0.1:8000/recipes`
- `http://127.0.0.1:8000/ingredients`
- `http://127.0.0.1:8000/stock`
- `http://127.0.0.1:8000/cart`
- `http://127.0.0.1:8000/to-buy`

Expected: each page loads with the dark theme and no server error.

- [ ] **Step 6: Commit**

```bash
git add Dockerfile docker-compose.yml README.md
git commit -m "chore: add local deployment docs"
```

---

## Self-Review Notes

- Spec coverage: the plan covers FastAPI, SQLite, dark server-rendered UI, Docker, ingredients, conversions, nested recipes, scaling, stock, cart, and To Buy.
- Deliberate first-pass scope: broad free-form unit parsing remains a follow-up from the spec and is not implemented here.
- Validation coverage: conversion and positive quantity rules are covered by service tests and database constraints; richer route-level error pages can be improved after the first working build.
- Type consistency: service tests and implementation use `Ingredient`, `IngredientConversion`, `Recipe`, `RecipeItem`, `StockItem`, `PlannedRecipe`, `ExpandedRecipe`, and `ShoppingLine` consistently.
