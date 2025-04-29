# Contributing to Linux Tutorials Repository

## How to Contribute

1. Fork the Repository

    Click the Fork button at the top of this page to create your own copy of the
    repository.

2. Clone Your Fork

    ```shell
    git clone https://github.com/<YOUR-USERNAME>/linux-tutorials.git
    cd linux-tutorials
    ```

3. Create a Branch

    Create a new branch for your contribution:

    ```shell
    git checkout -b fix-typo-in-ubuntu-install-docker
    ```

    Name your branch something descriptive, like `fix-typo-in-ubuntu-install-docker`
    or `add-new-kubernetes-tutorial`.

4. Make Your Changes

    - Write tutorials in clear, simple Markdown (`.md`) files.
    - Follow the existing style and structure when possible.
    - Use [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).
    - Keep language friendly, concise, and free of jargon whenever possible.
    - If applicable, add code examples and explanations.
    - Proofread for spelling, grammar, and formatting errors.

5. Test Your Changes

    Preview your Markdown to ensure correct formatting.  
    You can use GitHub’s built-in preview or a Markdown editor like
    [Visual Studio Code](https://code.visualstudio.com/).

6. Commit Your Changes

    > [!IMPORTANT]  
    > This repository uses
    > [semantic-release](https://semantic-release.gitbook.io).  
    > Please **follow**
    > **[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)**
    > when writing your commit messages.  
    > **Do not manually edit `CHANGELOG.md`**

    Commit your changes with a clear, descriptive message:

    ```shell
    git add .
    git commit -m "fix(ubuntu): Typo in Install Docker tutorial"
    ```

7. Push and Open a Pull Request

    Push your changes to your fork:

    ```shell
    git push origin fix-typo-in-ubuntu-install-docker
    ```

    Then open a Pull Request (PR) against the main branch.  
    Describe the changes you made and link to any related issues if applicable.

## Contribution Guidelines

- **Consistency**: Follow the style and tone used in existing tutorials.
- **Clarity**: Explain concepts as simply and clearly as possible.
- **Respect**: Be respectful in your interactions with others.

## Reporting Issues

Found an error or want to suggest an improvement?  
Please [open an issue](https://github.com/evan-kinney/linux-tutorials/issues/new)
with a clear description.
