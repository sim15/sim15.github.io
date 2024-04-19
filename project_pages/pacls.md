---
layout: page
title: Private Access Control for Function Secret Sharing
permalink: /projects/pacls
---

Function Secret Sharing (FSS; Eurocrypt 2015) allows a dealer to share a function $$f$$ with two or more evaluators. Given secret shares of a function $$f$$, the evaluators can locally compute secret shares of $$f(x)$$ on an input $$x$$, without learning information about $$f$$. In this paper, we initiate the study of access control for FSS. Given the shares of $$f$$, the evaluators can ensure that the dealer is authorized to share the provided function. For a function family $$\mathcal{F}$$ and an access control list defined over the family, the evaluators receiving the shares of $$f \in \mathcal{F}$$ can efficiently check that the dealer knows the access key for $$f$$. This model enables new applications of FSS, such as: anonymous authentication in a multi-party setting, access control in private databases, and authentication and spam prevention in anonymous communication systems. Our definitions and constructions abstract and improve the concrete efficiency of several re- cent systems that implement ad-hoc mechanisms for access control over FSS. The main building block behind our efficiency improvement is a discrete-logarithm zero-knowledge proof-of-knowledge over secret-shared elements, which may be of independent interest. We evaluate our constructions and show a $$50–70\times$$ reduction in computational overhead com- pared to existing access control techniques used in anonymous communication. In other applications, such as private databases, the processing cost of introducing access control is only $$1.5–3\times$$ when amortized over databases with 500,000 or more items.

Links:
- [PDF (full version)](https://eprint.iacr.org/2022/1707)
- [PDF (conference version)](https://ieeexplore.ieee.org/document/10179295)
- [Code](https://github.com/sachaservan/pacl)
- [Slides](https://sachaservanschreiber.com/slides/pacls.pdf)



Bib: 
```
@inproceedings{ServanSchreiber2023,
  title = {Private Access Control for Function Secret Sharing},
  url = {http://dx.doi.org/10.1109/sp46215.2023.10179295},
  doi = {10.1109/sp46215.2023.10179295},
  booktitle = {2023 IEEE Symposium on Security and Privacy (SP)},
  publisher = {IEEE},
  author = {Servan-Schreiber, Sacha and Beyzerov, Simon and Yablon, Eli and Park, Hyojae},
  year = {2023},
  month = may,
  pages = {809-828},
}
```