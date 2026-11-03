# Reducto cookbook

Examples and samples for [Reducto](https://reducto.ai) document parsing.

| Folder | What it holds |
|---|---|
| [`r-1-mini`](r-1-mini) | Reducto r-1 mini on AWS Marketplace: the endpoint contract, request and response samples, and a SageMaker notebook |

## Reducto r-1 mini on AWS Marketplace

Reducto r-1 mini is a page parser. You send one page image. The endpoint returns the layout blocks of the page: titles, text, tables, figures, and more, each with a position on the page.

r-1 mini is an Amazon SageMaker model package on AWS Marketplace. You subscribe to r-1 mini through Amazon SageMaker on AWS Marketplace. The model runs in your own AWS account.

- Start with [`r-1-mini/README.md`](r-1-mini/README.md) for the request and response contract.
- Deploy and call r-1 mini with [`r-1-mini/aws-sagemaker`](r-1-mini/aws-sagemaker).

## More

- Reducto documentation: https://docs.reducto.ai
- All samples in this repository use synthetic pages.

## License

This repository is licensed under the [Apache License 2.0](LICENSE).
