---
layout: post 
tags: 
  - sql 
title: SQL join visualization
---

<div>
    <style>
        .syntax {
            color: #E6153F;
        }
    </style>

    <div style="display: flex; flex-direction: row;height: 400px; width: 100%;">
        <canvas width="600" height="400" id="cycles">
            <p>Your browser does not support canvas =[</p>
        </canvas>
        <p id="query"
           style="vertical-align: middle;font-size: 30px; width: 100%; color: #222222">
        </p>
    </div>

    <script>
        color = {
            RED: '#ff66ff',
            GREEN: '#85e085',
            BLACK: '#003300',
            WHITE: '#FFFFFF'
        };
        var radius = 190;
        var lineWeight = 5;
        var leftCycle = {
            id: 1,
            y: 200,
            x: 200,
            text: "workers"
        };
        var rightCycle = {
            id: 2,
            x: 395,
            y: 200,
            text: "phones"
        };
        var middleCycle = {
            id: 3,
            leftX: leftCycle.x,
            leftY: leftCycle.y,
            rightX: rightCycle.x,
            rightY: rightCycle.y,
            leftStartAngle: Math.PI * 1.67,
            leftEndAngle: Math.PI / 3.5,
            rightStartAngle: Math.PI / 1.485,
            rightEndAngle: Math.PI * 1.33
        };
        const State = (function () {
            let instance;

            function key(element) {
                return element.id;
            }

            function createInstance() {
                const states = {};
                states.isSelected = function (element) {
                    return states[key(element)] === color.RED;
                };
                states.getColor = function (element) {
                    return states[key(element)];
                };
                states.changeColor = function (element) {
                    states[key(element)] = (states[key(element)] === color.RED) ? color.GREEN : color.RED;
                };
                states.init = function init() {
                    states[key(rightCycle)] = color.GREEN;
                    states[key(leftCycle)] = color.GREEN;
                    states[key(middleCycle)] = color.GREEN;
                };
                states.init();
                return states;
            }

            return {
                getInstance: function () {
                    if (!instance) {
                        instance = createInstance();
                    }
                    return instance;
                }
            };
        })();

        let canvas;
        let context;
        let state;

        function drawCycle(cycle) {
            context.beginPath();
            context.arc(cycle.x, cycle.y, radius, 0, 2 * Math.PI);
            context.fillStyle = state.getColor(cycle);
            context.fill();
            context.lineWidth = lineWeight;
            context.stokeStyle = color.BLACK;
            context.stroke();
        }

        function drawMiddleIntersect(element) {
            context.beginPath();
            context.arc(leftCycle.x, leftCycle.y, radius, element.leftStartAngle, element.leftEndAngle);
            context.arc(rightCycle.x, rightCycle.y, radius, element.rightStartAngle, element.rightEndAngle);
            context.fillStyle = state.getColor(element);
            context.fill();
            context.strokeStyle = color.BLACK;
            context.lineWidth = lineWeight;
            context.stroke();
        }

        function drawText() {
            context.font = '30px Arial';
            context.fillStyle = color.WHITE;
            context.fillText(leftCycle.text, leftCycle.x - 120, leftCycle.y);
            context.fillText(rightCycle.text, rightCycle.x + 30, rightCycle.y);
        }

        function calculate() {
            let text = "";
            if (state.isSelected(leftCycle) && state.isSelected(rightCycle) && state.isSelected(middleCycle)) {
                text = "<b class='syntax'>SELECT</b> * <b class='syntax'>" +
                    "FROM</b> workers <b class='syntax'>FULL OUTER JOIN</b> phones";
            } else if (state.isSelected(leftCycle) && state.isSelected(rightCycle)) {
                text = "<b class='syntax'>SELECT</b> * <b class='syntax'>" +
                    "FROM</b> workers <b class='syntax'>FULL OUTER JOIN</b> phones <b class='syntax'>" +
                    "WHERE</b> workers.type <b class='syntax'>IS NULL OR</b> phones.number <b class='syntax'>IS NULL</b>"
            } else if (state.isSelected(leftCycle) && state.isSelected(middleCycle)) {
                text = "<b class='syntax'>SELECT</b> * <b class='syntax'>" +
                    "FROM</b> workers <b class='syntax'>LEFT OUTER JOIN</b> phones";
            } else if (state.isSelected(middleCycle) && state.isSelected(rightCycle)) {
                text = "<b class='syntax'>SELECT</b> * <b class='syntax'>" +
                    "FROM</b> workers <b class='syntax'>RIGHT OUTER JOIN</b> phones"
            } else if (state.isSelected(middleCycle)) {
                text = "<b class='syntax'>SELECT</b> * <b class='syntax'>" +
                    "FROM</b> workers <b class='syntax'>INNER JOIN</b> phones"
            } else if (state.isSelected(leftCycle)) {
                text = "<b class='syntax'>SELECT</b> * <b class='syntax'>" +
                    "FROM</b> workers <b class='syntax'>LEFT OUTER JOIN</b> phones <b class='syntax'>" +
                    "WHERE</b> phones.number <b class='syntax'>IS NULL</b>"
            } else if (state.isSelected(rightCycle)) {
                text = "<b class='syntax'>SELECT</b> * <b class='syntax'>" +
                    "FROM</b> workers <b class='syntax'>RIGHT OUTER JOIN</b> phones <b class='syntax'>" +
                    "WHERE</b> workers.type <b class='syntax'>IS NULL</b>"
            }
            document.getElementById("query").innerHTML = text;
        }

        function intersect(x, y) {
            const rightDest = Math.sqrt(Math.pow(x - rightCycle.x, 2) + Math.pow(y - rightCycle.y, 2));
            const leftDest = Math.sqrt(Math.pow(x - leftCycle.x, 2) + Math.pow(y - leftCycle.y, 2));
            if (rightDest <= radius && leftDest <= radius) {
                state.changeColor(middleCycle);
            } else if (rightDest <= radius) {
                state.changeColor(rightCycle);
                drawCycle(rightCycle);
            } else if (leftDest <= radius) {
                state.changeColor(leftCycle);
                drawCycle(leftCycle);
            }
            drawMiddleIntersect(middleCycle);
            drawText();
            calculate();
        }

        function start() {
            canvas = document.getElementById("cycles");
            context = canvas.getContext("2d");
            state = State.getInstance();

            drawCycle(leftCycle);
            drawCycle(rightCycle);
            drawMiddleIntersect(middleCycle);
            drawText();
            canvas.addEventListener("click", function (event) {
                const rect = canvas.getBoundingClientRect();
                const x = event.clientX - rect.left;
                const y = event.clientY - rect.top;
                intersect(x, y)
            }, false);
        }

        start();
    </script>
</div>

The tables structure:

```sql
select *
from workers;
---------------
name | type
---------------
```

```sql
select *
from phones;
------------------
name | phone
------------------
```

[Read more on types of joins](https://izebit.ru/types-of-joins.html) 