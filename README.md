


1. Go to [editor](https://loyal--app.design.webflow.com/?mode=edit) and replace the "Loyal ads.txt" file by deleting the existing on and uploading the new file.
2. Copy asset URL to clipboard.  
3. Go to [Site Settings](https://webflow.com/dashboard/sites/loyal--app/general) > [Publishing](https://webflow.com/dashboard/sites/loyal--app/publishing)
4. Delete path for `/app-ads.txt`
5. Delete path for `/ads.txt`
6. **Publish* the site
6. Test `curl -sL https://www.loyal.app/ads.txt | grep 546926726`




## Fetch the current production version

curl -L https://www.loyal.app/ads.txt -o "./logs/$(date +%Y-%m-%d)-ads.txt"

## Check for changes

colordiff -u logs/$(date +%Y-%m-%d)-ads.txt "Loyal ads.txt" 


## Getting Started

brew install colordiff
