###Solved by bitvijays

Untitled-1.pdf
Misc (50 pts)
This PDF has a flag on it, but I can't find it... can you?

By reading other similar challenge's writeup, came to know about Predefined procedure sets and Text.

By parsing the pdf thru pdf-parser and grepping for Proc

```
pdf-parser Untitled-1_1a110935ec70b63ad09fec68c89dfacb\ \(1\).pdf | grep Proc
        /ProcSet [/PDF/Text/ImageC/ImageI]
        /ProcSet [/PDF/Text/ImageC/ImageI]
```

Then by using pdf2txt, found the flag

```
pdf2txt Untitled-1_1a110935ec70b63ad09fec68c89dfacb\ \(1\).pdf 
PCTF{how_2_pdf_yo}
```

The flag is **PCTF{how_2_pdf_yo}**
