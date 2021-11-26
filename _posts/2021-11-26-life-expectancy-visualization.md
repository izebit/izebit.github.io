---
layout: post
category: javascript
title: life expectancy visualization
---

<script type="application/javascript">
    function getTotalAge(country) {
        switch (country) {
            case "germany":
                return 82;
            case "russia":
                return 73;
            case "chad":
                return 53;
            default:
                return 70;
        }
    }

    function getCountOfDays(age) {
        return Math.round(age * 365.25);
    }

    function getBoxSize(canvasWidth, canvasHeight, boxCount) {
        const total = canvasWidth * canvasHeight;
        return Math.round(Math.sqrt(total / boxCount));
    }

    function getBoxColour(boxNumber, currentDays) {
        if (boxNumber <= currentDays)
            return '#c66570';
        else
            return '#8cc665';
    }


    function drawBox(canvasContext, boxNumber, boxSize, canvasWidth, boxColour) {
        const countOfColumns = Math.ceil(canvasWidth / boxSize);
        const rowNumber = Math.floor(boxNumber / countOfColumns);
        const columnNumber = Math.floor(boxNumber % countOfColumns)

        const padding = Math.max(Math.round(boxSize / 10) / 2, 1);
        canvasContext.fillStyle = boxColour;
        canvasContext.fillRect(
            (columnNumber * boxSize) + padding,
            (rowNumber * boxSize) + padding,
            boxSize - padding,
            boxSize - padding);
    }

    function draw(currentDays, totalDays) {
        const canvas = document.getElementById('canvas');

        if (canvas.getContext) {
            const context = canvas.getContext('2d');
            context.canvas.width = Math.round(innerWidth * 0.4);
            context.canvas.height = Math.round(innerHeight * 0.8);

            context.clearRect(0, 0, canvas.width, canvas.height);

            const {width, height} = canvas.getBoundingClientRect();
            const boxSize = getBoxSize(width, height, totalDays);

            for (let i = 0; i < totalDays; i++)
                drawBox(context, i, boxSize, width, getBoxColour(i, currentDays))
        }
    }

    function init(country, age) {
        const localStorage = window.localStorage;
        let currentAge;
        let currentCountry;
        if (country) {
            currentCountry = country;
            localStorage.setItem("current_country", currentCountry)
        } else {
            currentCountry = localStorage.getItem("current_country");
            const inputForm = document.getElementById("current_country");
            inputForm.value = currentCountry;        
        }

        if (age) {
            currentAge = age;
            localStorage.setItem("current_age", age)
        } else {
            currentAge = localStorage.getItem("current_age");
            const inputForm = document.getElementById("current_age");
            inputForm.value = currentAge;
        }

        console.log("country: " + currentCountry);
        console.log("age: " + currentAge);
        if (currentAge && currentCountry)
            draw(getCountOfDays(currentAge), getCountOfDays(getTotalAge(currentCountry)))
    }

    function selectCountry(country) {
        init(country, null)
    }

    function selectAge(age) {
        init(null, age)
    }

    window.onload = () => init(null, null);
</script>

<div style="
      display: flex;
      justify-content: center;
      margin: auto;
      width: 100%;
      padding-top: 10px;">

    <div>
        <canvas id="canvas"
                width="300" height="300">
        </canvas>
    </div>

    <div style="padding-left: 20px">
        <div style="padding-top: 10px">
            <label for="current_country">country where you live:</label>
            <form style="padding-top: 10px">
                <select id="current_country" onchange="selectCountry(this.value);">
                    <option value="russia" selected="selected">Russia</option>
                    <option value="germany">Germany</option>
                    <option value="chad">Chad</option>
                </select>
            </form>
        </div>
        <div style="padding-top: 10px">
            <label for="current_age">your age:</label>
            <input type="number" id="current_age"
                   name="current_age"
                   onchange="selectAge(this.value)"
                   style="
                            display:block;
                            width: 5em;
                            margin-top: 10px"
                   min="0" max="150">
        </div>
    </div>
</div>
