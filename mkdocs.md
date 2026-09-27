site_name: 图形学笔记
site_url: https://tao-mai.github.io/graphics-notes/
dev_addr: localhost:8001

theme:
  name: material
  variant: modern
  language: zh
  features:
    - navigation.tabs
    - navigation.tabs.sticky
    - navigation.sections

nav:
  - 首页: index.md
  - C++: cpp/index.md
  - 渲染:
      - 章节目录: chapters/index.md
      - 基础:
          - 零散: chapters/零散.md
          - IBL: chapters/IBL.md 
  - 仿真
  - 几何
  - 动画
  - UE

markdown_extensions:
  - pymdownx.arithmatex:
      generic: true
  - pymdownx.quotes:
      callouts: true

extra_javascript:
  - javascripts/mathjax.js
  - https://unpkg.com/mathjax@3/es5/tex-mml-chtml.js

extra_css:
  - stylesheets/extra.css
