# Spec draft

`0001-Propose-NFCIsoDepReader-...patch` is the proposed addition to the
[Web NFC Community Group report](https://w3c-cg.github.io/web-nfc/), as a patch against
[w3c-cg/web-nfc](https://github.com/w3c-cg/web-nfc) commit `5f29a90`. It adds the
`NFCIsoDepReader` chapter (API, application policies, the eMRTD policy, the scan and reading
algorithms), terminology, a use case, an example, threats and security policies.

To view it:

```sh
git clone https://github.com/w3c-cg/web-nfc.git && cd web-nfc
git checkout 5f29a90 && git am /path/to/this/repo/spec/*.patch
python3 -m http.server 8000   # open http://localhost:8000/index.html (ReSpec renders in the browser)
```

The same change will be proposed to the Community Group as a pull request.
