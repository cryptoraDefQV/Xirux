# Xurix File Hosting

A simple file hosting service that can be deployed on GitHub Pages and generates shareable links similar to catbox.moe.

## Features

- Drag and drop file uploads
- Direct file links in the format `https://yourusername.github.io/xurix/files/[random-id].[extension]`
- Discord-friendly embeds for media files
- Recent uploads history (saved in local storage)
- Mobile-responsive design

## Setup Instructions

### 1. Create a GitHub Repository

1. Go to [GitHub](https://github.com) and create a new repository named `xurix`
2. Clone the repository to your local machine:
   ```
   git clone https://github.com/yourusername/xurix.git
   cd xurix
   ```

### 2. Configure the HTML File

1. Download the `index.html` file from this repository
2. Open it in a text editor
3. Find the script section and modify these lines:
   ```javascript
   // Configuration - CHANGE THESE VALUES
   const GITHUB_USERNAME = 'yourusername'; // Your GitHub username
   const REPO_NAME = 'xurix'; // Your repository name
   ```
4. Replace `'yourusername'` with your actual GitHub username

### 3. Create the File Structure

1. Create a `files` directory in your repository
2. Add a placeholder file to ensure the directory is created on GitHub:
   ```
   mkdir files
   touch files/.gitkeep
   ```

### 4. Set Up GitHub Pages File Uploads

To enable actual file uploads, you need to create a GitHub Actions workflow:

1. Create a `.github/workflows` directory in your repository:
   ```
   mkdir -p .github/workflows
   ```

2. Create a file called `upload-file.yml` with the following content:

```yaml
name: Upload File

on:
  issues:
    types: [opened]

jobs:
  upload-file:
    runs-on: ubuntu-latest
    if: startsWith(github.event.issue.title, 'UPLOAD:')
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          
      - name: Install dependencies
        run: npm install axios

      - name: Download and save file
        run: |
          node -e "
            const axios = require('axios');
            const fs = require('fs');
            const path = require('path');
            
            async function downloadFile() {
              const issue = ${{ toJSON(github.event.issue) }};
              const title = issue.title;
              const body = issue.body;
              
              // Extract file URL from issue body
              const urlMatch = body.match(/URL: (https?:\/\/[^\s]+)/);
              if (!urlMatch) {
                console.error('No URL found in issue body');
                process.exit(1);
              }
              
              const url = urlMatch[1];
              const fileName = title.replace('UPLOAD:', '').trim();
              const fileExtension = path.extname(url).toLowerCase();
              const randomId = Math.random().toString(36).substring(2, 8);
              const finalFileName = randomId + fileExtension;
              const filePath = path.join('files', finalFileName);
              
              try {
                const response = await axios({
                  method: 'GET',
                  url: url,
                  responseType: 'stream'
                });
                
                const writer = fs.createWriteStream(filePath);
                response.data.pipe(writer);
                
                return new Promise((resolve, reject) => {
                  writer.on('finish', () => {
                    console.log('File downloaded to:', filePath);
                    resolve(finalFileName);
                  });
                  writer.on('error', reject);
                });
              } catch (error) {
                console.error('Error downloading file:', error);
                process.exit(1);
              }
            }
            
            downloadFile().then(fileName => {
              const issueComment = `File uploaded successfully: https://${process.env.GITHUB_REPOSITORY_OWNER}.github.io/${process.env.GITHUB_REPOSITORY.split('/')[1]}/files/${fileName}`;
              
              // Write to a file for the next step
              fs.writeFileSync('comment.txt', issueComment);
            });
          "
          
      - name: Commit file
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add files/
          git commit -m "Upload file from issue" || echo "No changes to commit"
          git push
          
      - name: Comment on issue
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            const issueComment = fs.readFileSync('comment.txt', 'utf8');
            
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: issueComment
            });
            
            await github.rest.issues.update({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'closed'
            });
```

### 5. Deploy to GitHub Pages

1. Commit all files to your repository:
   ```
   git add .
   git commit -m "Initial commit of Xurix file hosting"
   git push origin main
   ```

2. Enable GitHub Pages:
   - Go to your repository settings
   - Navigate to the "Pages" section
   - Select "Deploy from a branch" as the source
   - Choose the "main" branch and "/ (root)" folder
   - Click "Save"

3. Wait a few minutes for GitHub Pages to deploy your site

## How to Use

### For Visitors

1. Go to `https://yourusername.github.io/xurix/`
2. Upload a file by dragging it onto the drop zone or clicking the upload button
3. Once the upload completes, copy the direct link to share

### For File Uploads with GitHub Actions

Since GitHub Pages doesn't support server-side functionality, actual file uploads require a workaround using GitHub Actions:

1. On the GitHub repository page, create a new issue
2. Title the issue `UPLOAD: filename` (e.g., `UPLOAD: my-video`)
3. In the issue body, include `URL: https://example.com/path/to/file.mp4`
4. Submit the issue
5. The GitHub Action will:
   - Download the file from the URL
   - Save it to the `files` directory with a random ID
   - Comment on the issue with the direct link
   - Close the issue automatically

## Limitations

- GitHub Pages has a soft limit of 1GB per repository
- Individual files are limited to 100MB
- GitHub Actions may have rate limits
- Using issues for uploads is a workaround and not ideal for high-volume use

## Customization

- Edit the `index.html` file to change the appearance and functionality
- Modify the GitHub Actions workflow to add more features
- Add custom domains by configuring GitHub Pages settings

## License

This project is open source and available under the [MIT License](LICENSE).
