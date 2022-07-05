---
title: "10 weeks challenge - day 01"
date: 2022-07-03T20:00:00-07:00
draft: true
toc: false
images:
tags:
  - untagged
---

### Overview

I'm starting a web development challenge with Nomadcoders(nomadcoders.co). This 10 weeks challenge is to review my JS knowledges and to start bootstrapping my ideas. I don't think this would cover everything, but I'm just hoping that I can meet some people who can talk about programming and have fun.

### Today I learned (reviewed)

- basic understanding on HTML, CSS and JS
- environment setup

### Resources

- [kokoa clone 1.5 - 1.9, 2.0 - 2.2](https://nomadcoders.co/kokoa-clone)
- [React JS](https://nomadcoders.co/react-masterclass/)
- prettier (VSCode extension)
- material icon theme (VSCode extension)
- community material theme (VSCode extension)

### Challenge

- quiz about basic concepts of HTML and CSS

### Additional learning

- look through React basic course

  - state : variable(or property) that is updating when the binded callback function is called (I'm not sure the 'binding' is a right way to explain it)
  - props : passed object that is containing attributes from user-defined component
  - function based components : seems like it is newer way to make React Components than Class based components. It is creating components as function with useEffect, useState and etc, instead of inheriting `React.Component`.
  - useEffect
  - useState

  - npx create-react-app : CRA command
  - npm start : running local server

  - styled components : there are 3 other ways to use CSS in React, but `styled components` library (https://styled-components.com/) is the best way to write css codes for React components. (https://styled-components.com/docs/basics)
    - configurable (with props): styled components can get props
    ```
    ${(props) => props.bgColor};
    ```
    - extend(by inheriting): styled(styleComponent)
    - extend : `as` props
    - styled.tag.attrs({required:true})``;
    - import {keyframes} from "styled-components"
    - ${styled-component} pseudo selector
    - themes : import {ThemeProvider} from "styled-components";
