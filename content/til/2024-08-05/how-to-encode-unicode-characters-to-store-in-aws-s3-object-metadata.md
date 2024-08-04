---
title: 'How to encode Unicode characters to store in AWS S3 object metadata'
date: '2024-08-05T09:00:00+10:00'
---

I recently ran into [this error](https://github.com/boto/botocore/blob/1.34.130/botocore/handlers.py#L629-L638) uploading images to AWS S3 using the `boto3` package.

> Parameter validation failed:  
> Non ascii characters found in S3 metadata for key "filename", value: "ACME™ Anvil.jpg".  
> S3 metadata can only contain ASCII characters.

The character in question was ™. Here’s a simplified example of the code.

```python
filename = "ACME™ Anvil.jpg"
metadata: dict[str, str] = {
    "filename": filename,
}
client.upload_fileobj(
    Fileobj=...,
    Bucket=...,
    Key=...,
    ExtraArgs={"Metadata": metadata}
)
```

I wanted to preserve the original filename, so I learnt to use `backslashreplace` when encoding the filename to ASCII. Note, I have to subsequently use `.decode()` as both metadata keys and values must be `str`, not `bytes`.

```python
metadata: dict[str, str] = {
    "filename": filename.encode("ascii", "backslashreplace").decode(),
}
```

This results in the original filename being stored as such.

```python
>>> filename.encode("ascii", "backslashreplace").decode()
'ACME\\u2122 Anvil.jpg'
```

And the following can be used to reverse the result at a later date.

```python
>>> "ACME\\u2122 Anvil.jpg".encode("ascii").decode("unicode-escape")
'ACME™ Anvil.jpg'
```

At first, I thought this didn’t work for all Unicode characters, namely [emoji zwj sequences](https://www.unicode.org/emoji/charts/emoji-zwj-sequences.html) which use `U+200D` to join the characters into a single glyph, for example.

```python
>>> "😵‍💫".encode("ascii", "backslashreplace").decode()
'\\U0001f635\\u200d\\U0001f4ab'

>>> "\\U0001f635\\u200d\\U0001f4ab".encode("ascii").decode("unicode-escape")
'😵\u200d💫'
```

Thanks to [Lawrence Hudson](https://github.com/quicklizard99) he pointed out that it was the REPL showing the [repr](https://docs.python.org/3/library/functions.html#repr) of the result rather than the Unicode string. Using `print()` on the result highlighted this.

```python
>>> print("\\U0001f635\\u200d\\U0001f4ab".encode("ascii").decode("unicode-escape"))
😵‍💫
```
