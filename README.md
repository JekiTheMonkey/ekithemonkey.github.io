start locally:
> bundle exec jekyll serve

# syntax:

## comparison-slider (should use ordinary quotes on filepaths):

first_image_description and second_image_description is not required (by default its Before/After)
description by default is true (do not use quotas)
```
{%
  include comparison-slider.html
  first_image='assets/images/filename.png'
  second_image='assets/images/filename2.png'
  description=false
  first_image_description='hello'
  second_image_description='hey'
%}
```

