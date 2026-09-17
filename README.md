# Articles

Public articles, technical notes, research, and independent analysis.

This repository is a collection of public-facing writing on software engineering, Apple platform development, AI and technology, and broader research topics.

## Apple Platforms

### Diagnosing a Trailing `UINavigationBar` Animation in SwiftUI

From a one-point visual glitch to a navigation lifecycle problem: separating SwiftUI layout state, UIKit navigation-bar lifecycle, and Core Animation presentation state to isolate a subtle iPhone push-transition artifact.

- [English: Diagnosing a Trailing `UINavigationBar` Animation in SwiftUI](./apple-platforms/diagnosing-swiftui-uinavigationbar-trailing-animation.md)
- [中文：SwiftUI `UINavigationBar` Push 后约 1pt 横向抖动：原因与解决方案](./apple-platforms/zh/diagnosing-swiftui-uinavigationbar-trailing-animation.md)

## Repository Structure

Articles are organized by topic first. When a Chinese version is available, it uses the same filename under that category's `zh/` directory.

```text
articles/
├── README.md
└── apple-platforms/
    ├── diagnosing-swiftui-uinavigationbar-trailing-animation.md
    └── zh/
        └── diagnosing-swiftui-uinavigationbar-trailing-animation.md
```

The repository root `README.md` is the primary index for all published articles.
