+++
type = "post"
title = "TIL huggingface-cli delete-cache to clean local models"
date = 2025-05-06
+++

I’m currently working on a project that made me download a bunch of models from Hugging Face to compare their performance. At some point, I wanted to clean up my Hugging Face cache and delete the ones that clearly weren’t working for my use case.

At first, I was just listing the models in my cache directory (by default it’s `.cache/huggingface/hub`) and manually deleting folders one by one.

Then I found out about [`huggingface-cli delete-cache`](https://huggingface.co/docs/huggingface_hub/v0.30.2/guides/manage-cache#clean-your-cache). It lets you interactively pick which models to delete, shows when you last used them, and tells you how much space they take up. There's also a [`scan-cache`](https://huggingface.co/docs/huggingface_hub/v0.30.2/guides/manage-cache#scan-your-cache) command if you just want to check what’s in there without deleting anything.

Another nice detail: it shows different revisions of a model, so you can delete just the ones you don’t need instead of wiping the whole thing.

I don’t know all the details yet, but this definitely feels like a better—and probably safer—way to clean your Hugging Face cache. You can read more about the Hugging Face cache in the [documentation](https://huggingface.co/docs/huggingface_hub/v0.30.2/guides/manage-cache#understand-caching).
