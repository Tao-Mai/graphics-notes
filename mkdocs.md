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
      - 章节目录: 渲染/index.md
      - 基础:
          - 零散: 渲染/零散.md
          - IBL: 渲染/IBL.md
      - 采样:
          - 低差异化序列: 渲染/采样/低差异化序列.md
      - 蒙特卡洛积分:
          - 概述: 渲染/蒙特卡洛积分/概述.md

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
