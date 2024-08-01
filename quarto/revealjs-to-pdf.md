To render a reveal js file to pdf, that is similar in formatting use the following command:

```bash
docker run --rm -t -v `pwd`:/slides -v $(pwd):/home/user astefanutti/decktape --screenshots-size 1920x1080 /home/user/PRESENTATIONNAME.html slides.pdf
```

Run the above command where you have the `PRESENTATIONNAME.html` quarto rendered revealjs format file.

source link: https://github.com/quarto-dev/quarto-cli/discussions/7018#discussioncomment-7897967
