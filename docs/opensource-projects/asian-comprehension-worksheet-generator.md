# Asian Comprehension Worksheet Generator

![sample worksheet](../assets/opensource-projects/asian-comprehension-worksheet-generator/sample.png)

## Background
This is a similar project to [Asian Character Worksheet Generator](https://github.com/januschung/asian-character-worksheet-generator).

As an immigrant and father of two boys living in America, I want teach my sons comprehension and reading skills of my language. I want to pick my own material which fits their levels.  

## Benefit of the project
The worksheet comes with empty column next to each word column. You can add phonetic note besides the character or use it to practice writing.

## Requirements
1. [python3](https://www.python.org/downloads/)
2. [pip](https://pip.pypa.io/en/stable/installing/)

Run the following to install required packages
``` shell
pip install -r requirements.txt
```

## How to Use
1. Edit the file `text` with the content of your choice (or use the sample).
``` yaml
光輝歲月
鐘聲響起歸家的信號，
在他生命裡，彷彿帶點唏噓。
黑色肌膚給他的意義，
是一生奉獻，膚色鬥爭中。
年月把擁有變做失去，
疲倦的雙眼帶著期望。
今天只有殘留的軀殼，
迎接光輝歲月，
風雨中抱緊自由。
一生經過傍徨的掙扎，
自信可改變未來，
問誰又能做到。
可否不分膚色的界限，
願這土地裡，不分你我高低。
繽紛色彩閃出的美麗，
是因它沒有，分開每種色彩。
``` 
2. Generate the worksheet in pdf format with the following command:
``` shell
python3 run.py
```
3. Print out the generated file `worksheet.pdf`

## Language Supported
1. Chinese
2. Japanese (Hiragana and Katagana)

## Sample
[sample worksheet](https://github.com/januschung/asian-comprehension-worksheet-generator/blob/master/sample-worksheet.pdf)

## Code Overview
Everything is written in python in `run.py`. You can play with the font and grid size with the variables under the `# Basic settings` section.

## Contributing
I appreciate all suggestions or PRs which will help kids learn Asian language better. Feel free to fork the project and create a pull request with your idea.

## Special Thanks
My lovely sons Tim and Hin. 