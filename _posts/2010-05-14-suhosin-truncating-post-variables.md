---
published: true
layout: default
---

## PHP Suhosin truncating POST variables
I recently spent an afternoon trying to figure out why my POST-data was being truncated on our production server.

I was sending a pretty large POST from a large form (approx. 1200 input fields) but for some reason it was being truncated. After quite a while I found the culprit:

  suhosin.post.max_vars = 200
  suhosin.request.max_vars = 200

Setting these two two a higher number solved the problem for me. Hopefully this saves you from spending the same time debugging this problem as I did.
