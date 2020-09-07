<template>
  <div>
    <div class="grids">
      <div class="player-side">
        <h2>Player: {{ player.numOfLiveShips }}</h2>
        <div class="player-grid">
          <div
            class="place"
            v-for="(_, i) in Math.pow(MAX + 1, 2)"
            :key="keys[MIN + i]"
            :data-player-coord="stringifiedCoords[MIN + i]"
          ></div>
          <div
            v-for="(ship, index) in player.ships"
            :id="index"
            :key="index"
            :data-length="ship.length"
            :data-direction="ship.direction"
            :style="{
                            top: ship.position[0].y * 50 + 'px',
                            left: ship.position[0].x * 50 + 'px',
                            width: (ship.direction == 'horizontal') ? (ship.length * 50  + 'px') : '50px',
                            height: (ship.direction == 'vertical') ? (ship.length * 50  + 'px') : '50px',
                            opacity: '0.7',
                        }"
            class="ship"
            :class="{
                            gridSnap: gameFlow.placingShips,
                            moveCursor: gameFlow.placingShips,
                        }"
          ></div>
        </div>
      </div>
      <div class="comp-side">
        <h2>Computer: {{ computer.numOfLiveShips }}</h2>
        <div class="computer-grid">
          <div
            class="square"
            v-for="(_, i) in Math.pow(MAX + 1, 2)"
            :key="keys[MIN + i]"
            :data-coord="stringifiedCoords[MIN + i]"
          ></div>
        </div>
      </div>
    </div>
    <div>
      <button v-if="!gameFlow.gameStarted" @click="gameFlow.gameStarted = true">Start Game</button>
    </div>
  </div>
</template>

<script>
import interact from "interactjs";
export default {
  props: {
    gameFlow: {
      type: Object,
      required: true,
    },
    player: {
      type: Object,
      required: true,
    },
    computer: {
      type: Object,
      required: true,
    },
    createShip: {
      type: Function,
    },
    shipFactory: {
      type: Function,
    },
  },
  data() {
    return {
      MIN: 0,
      MAX: 9,
      compShipsAlive: 10,
      playerShipsAlive: 10,
    };
  },

  mounted() {
    // for (let y = this.MIN; y <= this.MAX; y++) {
    //   for (let x = this.MIN; x <= this.MAX; x++) {
    //     if (this.player.gridMap[y][x] != "") {
    //       let coordShip = JSON.stringify({ x: x, y: y });
    //       let shipSquare = document.querySelector(
    //         `[data-player-coord='${coordShip}']`
    //       );

    //       // shipSquare.textContent = "O";
    //       // shipSquare.style.fontSize = 20 + "px";
    //       shipSquare.style.backgroundColor = "#2ecc71";
    //     }
    //   }
    // }

    const that = this;
    let x = 0;
    let y = 0;
    let newCoords;
    let eventTarget;

    interact(".gridSnap")
      .draggable({
        modifiers: [
          interact.modifiers.snap({
            targets: [interact.createSnapGrid({ x: 50, y: 50 })],
            range: Infinity,
            relativePoints: [{ x: 0, y: 0 }],
            offset: "parent",
          }),
          interact.modifiers.restrict({
            restriction: "parent",
            elementRect: { top: 0, left: 0, bottom: 1, right: 1 },
            endOnly: true,
          }),
        ],
        inertia: false,
      })
      .on("dragmove", function (event) {
        const length = event.target.getAttribute("data-length");
        const direction = event.target.getAttribute("data-direction");
        x += event.dx;
        y += event.dy;
        eventTarget = event.target;

        event.target.style.webkitTransform = event.target.style.transform =
          "translate(" + x + "px, " + y + "px)";
        newCoords = [
          Math.round(
            (parseInt(event.target.style.top.replace("px", "")) + y) / 50
          ).toString() +
            Math.round(
              (parseInt(event.target.style.left.replace("px", "")) + x) / 50
            ).toString(),
        ];

        for (let i = 1; i < length; i++) {
          if (length === 1) return;
          if (direction === "horizontal") {
            newCoords.push(
              Math.round(
                (parseInt(event.target.style.top.replace("px", "")) + y) / 50
              ).toString() +
                Math.round(
                  (parseInt(event.target.style.left.replace("px", "")) + x) /
                    50 +
                    i
                ).toString()
            );
          } else if (direction === "vertical") {
            newCoords.push(
              Math.round(
                (parseInt(event.target.style.top.replace("px", "")) + y) / 50 +
                  i
              ).toString() +
                Math.round(
                  (parseInt(event.target.style.left.replace("px", "")) + x) / 50
                ).toString()
            );
          }
        }
      })
      .on("dragend", function () {
        const shipID = eventTarget.getAttribute("id");
        const isVertical =
          eventTarget.getAttribute("data-direction") == "vertical"
            ? true
            : false;
        const shipLength = Number(eventTarget.getAttribute("data-length"));
        const oldX = Number(newCoords[0][1]) - Math.round(x / 50);
        const oldY = Number(newCoords[0][0]) - Math.round(y / 50);
        const newX = Number(newCoords[0][1]);
        const newY = Number(newCoords[0][0]);
        console.log(that.player.gridMap);
        if (shipLength === 1) {
          that.player.gridMap[oldY][oldX] = "";
        } else if (!isVertical) {
          for (let i = 0; i < shipLength; i++) {
            that.player.gridMap[oldY][oldX + i] = "";
            console.log(oldX + i);
          }
        } else if (isVertical) {
          for (let i = 0; i < shipLength; i++) {
            that.player.gridMap[oldY + i][oldX] = "";
          }
        }

        that.createShip(
          newY,
          newX,
          that.shipFactory(shipLength),
          isVertical,
          that.player.gridMap,
          shipID
        );
        console.log(that.player.gridMap);
        console.log(shipID, isVertical, shipLength, { y, x });

        eventTarget.style.webkitTransform = eventTarget.style.transform =
          "translate(" + x + "px, " + y + "px)";

        x = 0;
        y = 0;
      });
  },
  computed: {
    coords() {
      const coords = [];
      for (let i = this.MIN; i <= this.MAX; i++) {
        for (let j = this.MIN; j <= this.MAX; j++) {
          coords.push({ x: j, y: i });
        }
      }
      return coords;
    },
    keys() {
      return this.coords.map((c, i) => `${JSON.stringify(c)}-${i}`);
    },
    stringifiedCoords() {
      return this.coords.map((c) => JSON.stringify(c));
    },
  },
};
</script>

<style>
.grids {
  display: flex;
  justify-content: space-around;
}

.square {
  font-size: 0;
  text-align: center;
  line-height: 50px;
  border: 1px solid black;
}

.computer-grid {
  display: grid;
  grid-template-rows: repeat(10, 50px);
  grid-template-columns: repeat(10, 50px);
}

.fog {
  z-index: 10;
  background-color: aquamarine;
}

.place {
  font-size: 0;
  text-align: center;
  line-height: 50px;
  border: 1px solid black;
}

.player-grid {
  display: grid;
  grid-template-rows: repeat(10, 50px);
  grid-template-columns: repeat(10, 50px);
  position: relative;
}

#grid-snap {
  width: 40%;
  background-color: #29e;
  color: #fff;
  font-size: 1.2em;
  border-radius: 4px;
  padding: 2%;
  margin: 5%;
  touch-action: none;
}

.gridCell {
  box-shadow: rgb(180, 180, 255) 1px 0px 0px 0px,
    rgb(180, 180, 255) 0px 1px 0px 0px, rgb(180, 180, 255) 1px 1px 0px 0px,
    rgb(180, 180, 255) 1px 0px 0px 0px inset,
    rgb(180, 180, 255) 0px 1px 0px 0px inset;
  touch-action: none;
}

.ship {
  position: absolute;
  border: 2px solid rgb(0, 0, 255);
  box-sizing: border-box;

  background-color: rgba(0, 0, 255, 0.05);
  z-index: 4;
  touch-action: none;
}

.moveCursor {
  cursor: move;
}
</style>
