### **Part 1: Creating a New Repository on GitHub**

First, you'll need to create a new, empty repository on GitHub. This will serve as the remote location for your code.

1.  **Sign in to GitHub:** Go to [GitHub](https://github.com) and log in to your account.
2.  **Create a New Repository:** In the upper-right corner of any page, click the **+** icon, and then select **New repository**.
3.  **Repository Details:**
    *   **Repository name:** Choose a name for your repository. For example, "my-awesome-project".
    *   **Description (optional):** You can add a brief description of your project.
    *   **Visibility:** You can choose to make your repository **Public** (visible to everyone) or **Private** (you choose who can see and commit).
    *   **Initialize this repository with a README:** It is recommended to *not* initialize the repository with a README, .gitignore, or license if you are importing an existing project. You can add these files later.
4.  **Create Repository:** Click the "Create repository" button. You will be redirected to the new repository's page, which will display a URL. Keep this URL handy.

### **Part 2: Uploading Your Local Code**

Now, you will use the command line to "push" your local code to the newly created GitHub repository.

**Prerequisites:**
*   You need to have **Git** installed on your computer. You can check if it's installed by opening a terminal or command prompt and typing `git --version`. If not, you can download it from the [official Git website](https://git-scm.com/downloads).
*   It's also recommended to configure your Git username and email if you haven't already. You can do this by running the following commands in your terminal, replacing the placeholder text with your information:
    ```bash
    git config --global user.name "Your Name"
    git config --global user.email "your.email@example.com"
    ```

**Steps:**

1.  **Open a Terminal or Command Prompt:** Navigate to the root folder of your code on your computer using the `cd` command. For example:
    ```bash
    cd path/to/your/project
    ```

2.  **Initialize a Git Repository:** If your project is not already a Git repository, you need to initialize it.
    ```bash
    git init
    ```
    This command creates a hidden `.git` subdirectory in your project folder, which will track all the changes.

3.  **Add Your Files to the Staging Area:** You need to tell Git which files you want to track. To add all the files in your project folder, use the following command:
    ```bash
    git add .
    ```
    The `.` represents all files and folders in the current directory.

4.  **Commit Your Files:** A commit is like a snapshot of your staged files at a specific point in time.
    ```bash
    git commit -m "Initial commit"
    ```
    The `-m` flag allows you to add a commit message directly from the command line.

5.  **Connect Your Local Repository to the Remote GitHub Repository:** You need to tell your local Git repository where to push the code. Use the URL you copied after creating the repository on GitHub.
    ```bash
    git remote add origin <repository_url>
    ```    Replace `<repository_url>` with the actual URL of your GitHub repository. It should look something like `https://github.com/your-username/your-repository-name.git`.

6.  **Push Your Code to GitHub:** Finally, you can push your committed files to the remote repository.
    ```bash
    git push -u origin main
    ```
    *   `origin` is the default name for your remote repository.
    *   `main` is the name of the branch you are pushing. Some older repositories might use `master` as the default branch name.
    *   The `-u` flag sets the upstream (remote) branch for your local branch. This allows you to use `git pull` and `git push` in the future without specifying the remote and branch.
