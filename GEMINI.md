# Project Overview

This is a Jekyll-based personal blog site using the `jekyll-text-theme`. The site is configured through `_config.yml` and includes blog posts in the `_posts` directory. The visual appearance is controlled by SCSS files in `_sass` and layouts in `_layouts`.

# Building and Running

This project uses both Ruby (for Jekyll) and Node.js (for development scripts). Docker is also configured to provide a consistent development environment.

## Key Commands

*   **Install Dependencies:**
    *   For Ruby gems: `bundle install`
    *   For Node.js packages: `npm install`

*   **Run the development server:**
    ```bash
    npm run serve
    ```
    This will start a local server, and you can view the site at `http://localhost:4000`.

*   **Build the site for production:**
    ```bash
    npm run build
    ```
    This will generate the static site in the `_site` directory.

*   **Using Docker:**
    The project includes several Docker configurations for development and production. Refer to the `scripts` section in `package.json` for a full list of Docker commands. For example, to run the development server in Docker:
    ```bash
    npm run docker-dev:dev
    ```

# Development Conventions

*   **Linting:** The project uses ESLint for JavaScript and Stylelint for SCSS.
    *   `npm run eslint`: To lint JavaScript files.
    *   `npm run stylelint`: To lint SCSS files.
*   **Git Hooks:** There is a pre-commit hook configured with Husky that runs `commitlint` to ensure conventional commit messages.
*   **Blog Posts:** New blog posts should be created in the `_posts` directory, following the Jekyll naming convention `YYYY-MM-DD-title.md`.
