# Recipe Site Design

## Goal

Build a single-user local recipe website that uses a relational data model for recipes, ingredients, stock, planned cooking, and computed shopping needs. The first version should prioritize accurate recipe scaling, reusable sub-recipes, unit conversion through canonical ingredient data, and practical web forms for entering recipes.

## Constraints

- Single-user local app for now; no login or account system.
- Stack: FastAPI, SQLite, server-rendered HTML, CSS, and small targeted JavaScript.
- Explicit dark-mode UI with high contrast.
- Dockerized local deployment with a persistent SQLite database volume.
- Nginx is optional later for local-network access and is not required for the first implementation.

## Pages

- Recipes: browse recipes, view a recipe, create/edit recipes, manage ingredients and sub-recipes, and scale by yield or item quantity.
- Stock: manage current ingredient quantities.
- Cart: select recipes/sub-recipes to cook, each with its own target yield.
- To Buy: compute shortages by subtracting stock from the expanded cart totals.

## Architecture

FastAPI serves Jinja-rendered HTML pages and form endpoints. SQLite stores application data. The backend owns recipe expansion, conversion math, scaling, validation, and shopping list calculation. JavaScript is limited to dynamic form rows, scale-preview interactions, and UI controls that are awkward with plain HTML forms.

The app should keep clear service boundaries:

- Unit conversion service: converts ingredient-specific units to grams and formats amounts for display.
- Recipe expansion service: expands nested recipes and computes totals for full trees or subtrees.
- Planning service: expands the cart and computes shopping shortages against stock.
- Form/view handlers: validate requests, call services, and render templates.

## Data Model

### Ingredients

`ingredients` stores canonical food items. Each ingredient normalizes quantities to grams internally. Ingredients also store preferred display units for stock and shopping, since recipe rows can choose their own display unit.

Global weight units such as grams, kilograms, ounces, and pounds have fixed conversions and should not be duplicated per ingredient.

`ingredient_conversions` stores ingredient-specific conversions into grams. This includes:

- Volume conversions based on density, such as tablespoons or cups of olive oil.
- Count/package conversions, such as `1 clove garlic = 5 g`, `1 bunch cilantro = 60 g`, `1 can coconut milk = 382 g`, or `1 carton eggs = 12 each` with each then mapped into grams where needed.

For v1, recipe and stock entries should use controlled units and require enough conversion data to convert the entry to grams. Broad automatic parsing from many free-form input styles is an early follow-up, not required in the first pass.

### Recipes

`recipes` represents both top-level recipes and reusable sub-recipes. A recipe has:

- Name.
- Base yield quantity.
- Base yield unit.
- Free-form instructions.

`recipe_items` stores ordered rows inside a recipe. Each row is either:

- A direct ingredient amount, with ingredient, quantity, unit, recipe display unit, prep/note, and sort order.
- A sub-recipe reference, with referenced recipe, included amount, unit, note, and sort order.

Sub-recipes are included by measured output, such as `1.5 cups pesto`, `500 g sauce`, or `15 each naan`. The referenced recipe's base yield defines how to scale its internal ingredient tree.

Duplicate ingredients can appear multiple times in a recipe for readability, such as separate butter rows for different uses. Recipe views should preserve separate rows, while totals views can combine duplicate ingredients.

## Recipe Math

Every ingredient amount converts to grams using the ingredient's conversion rules. A recipe can be scaled by:

- Target yield, such as `8 servings`.
- A direct ingredient or sub-recipe item quantity.

Scaling computes a scale factor against the recipe's base quantities. Recipe totals can be shown for:

- The full recipe tree.
- Any subtree, such as just the pesto inside pesto pasta.
- Combined ingredient totals across duplicates.
- Original row structure for readability.

## Stock And Shopping

`stock_items` stores ingredient, quantity, unit, optional location, and optional note. Stock quantities convert to grams through the same conversion service as recipes.

`planned_recipes` stores recipes selected for the cart, each with target yield quantity, target yield unit, and optional note.

The To Buy page expands all planned recipe trees, combines ingredient totals in grams, subtracts stock totals in grams, and displays remaining shortages using each ingredient's shopping preferred unit. The first version of To Buy is read-only/computed. Actions like marking items bought or moving them into stock are later enhancements.

## Validation

- Recipe ingredient rows must reference a known ingredient.
- Entered units must be controlled units.
- Ingredient/unit combinations must be convertible to grams before they can participate in recipe, stock, or shopping calculations.
- Sub-recipe references must avoid cycles.
- Recipe yields must be positive and convertible where parent recipes reference them.
- Planned recipe target yields must be positive.

## Testing

Automated tests should cover:

- Ingredient-specific unit conversion to grams.
- Package/count conversions.
- Recipe scaling by yield.
- Recipe scaling by a direct item quantity.
- Nested recipe expansion.
- Full-tree and subtree totals.
- Duplicate ingredient rollups.
- Stock subtraction.
- Basic page/form flows for creating recipes, stock, cart entries, and viewing the To Buy list.

## Follow-Ups

- Automatic unit parsing from varied free-form input, such as pasted recipe lines.
- More advanced shopping actions, such as marking items bought or moving purchases into stock.
- Nginx/local-network polish.
- Structured instructions, timers, photos, tags, search filters, and meal history.
