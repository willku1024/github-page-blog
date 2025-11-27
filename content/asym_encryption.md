+++
title = "Asymmetric Encryption Scenarios"
template = "page.html"
date = 2025-11-27T18:31:19
[taxonomies]
tags = ['Networking & Security']
[extra]
+++


| Statement                                                   | Which key to use (Public or Private) | Whose key to use (Jinia or Bill) |
| :---------------------------------------------------------- | :----------------------------------- | :------------------------------- |
| Jinia wants to send the encrypted message to the Bill       | **Public**                           | **Bill**                         |
| Bill wants to read an encrypted message sent by the Jinia   | **Private**                          | **Bill**                         |
| Jinia receives an encrypted reply message from Bill         | **Private**                          | **Jinia**                        |
| Jinia wants to send Bill a message with a digital signature | **Private**                          | **Jinia**                        |
| Bill wants to see Jinia's digital signature                 | **Public**                           | **Jinia**                        |
