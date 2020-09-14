<template>
    <div>
        <div class="grids">
            <div class="player-side">
                <h2>Player: {{ playerShipsAlive }}</h2>
                <div class="player-grid" :class="{ active: !gameFlow.playerTurn && gameFlow.gameStarted }">
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
                        :data-coords="ship.position[0].y.toString() + ship.position[0].x.toString()"
                        :style="{
                            top: ship.position[0].y * 50 + 'px',
                            left: ship.position[0].x * 50 + 'px',
                            width: ship.direction == 'horizontal' ? ship.length * 50 + 'px' : '50px',
                            height: ship.direction == 'vertical' ? ship.length * 50 + 'px' : '50px',
                            opacity: '0.7',
                        }"
                        class="ship"
                        :class="{
                            gridSnap: gameFlow.placingShips,
                            moveCursor: gameFlow.placingShips,
                        }"
                        @dblclick="rotateShip"
                    ></div>
                </div>
            </div>
            <div class="comp-side">
                <h2>Computer: {{ compShipsAlive }}</h2>
                <div class="computer-grid" :class="{ active: gameFlow.gameStarted && gameFlow.playerTurn }">
                    <div
                        class="square"
                        v-for="(_, i) in Math.pow(MAX + 1, 2)"
                        :key="keys[MIN + i]"
                        :data-coord="stringifiedCoords[MIN + i]"
                        @click.once="playerAttack"
                    ></div>
                </div>
            </div>
        </div>
        <div>
            <button
                v-if="!gameFlow.gameStarted"
                @click="
                    gameFlow.gameStarted = true;
                    gameFlow.playerTurn = true;
                    gameFlow.placingShips = false;
                "
            >
                Start Game
            </button>
            <button
                v-if="gameFlow.gameStarted"
                @click="
                    gameFlow.gameStarted = false;
                    gameFlow.placingShips = true;
                    gameFlow.winner = false;
                    gameFlow.playerTurn = false;
                    $emit('resetGame');
                "
            >
                Reset Game
            </button>
        </div>
    </div>
</template>

<script>
import interact from 'interactjs';
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
        isCoordValid: {
            type: Function,
        },
    },
    data() {
        return {
            MIN: 0,
            MAX: 9,
            compShipsAlive: this.computer.numOfLiveShips,
            playerShipsAlive: this.player.numOfLiveShips,
            compAttackOptions: [],
            lastSuccesfullAttack: [],
        };
    },

    mounted() {
        // for (let y = this.MIN; y <= this.MAX; y++) {
        //     for (let x = this.MIN; x <= this.MAX; x++) {
        //         // for player:          if (this.player.gridMap[y][x] != "") {

        //         if (Array.isArray(this.computer.gridMap[y][x])) {
        //             let coordShip = JSON.stringify({ x: x, y: y });
        //             let shipSquare = document.querySelector(
        //                 // for player: data-player-coord
        //                 `[data-coord='${coordShip}']`
        //             );

        //             shipSquare.style.backgroundColor = '#2ecc71';
        //         }
        //     }
        // }

        //generate options for computer attack
        for (let i = 0; i < 100; i++) {
            this.compAttackOptions.push(String(i).padStart(2, '0'));
        }

        const that = this;
        let x = 0;
        let y = 0;
        let newCoords;
        let eventTarget;

        interact('.gridSnap')
            .draggable({
                modifiers: [
                    interact.modifiers.snap({
                        targets: [interact.createSnapGrid({ x: 50, y: 50 })],
                        range: Infinity,
                        relativePoints: [{ x: 0, y: 0 }],
                        offset: 'parent',
                    }),
                    interact.modifiers.restrict({
                        restriction: 'parent',
                        elementRect: { top: 0, left: 0, bottom: 1, right: 1 },
                        endOnly: true,
                    }),
                ],
                inertia: false,
            })
            .on('dragmove', function(event) {
                const length = event.target.getAttribute('data-length');
                const direction = event.target.getAttribute('data-direction');
                x += event.dx;
                y += event.dy;
                eventTarget = event.target;

                event.target.style.webkitTransform = event.target.style.transform =
                    'translate(' + x + 'px, ' + y + 'px)';
                newCoords = [
                    Math.round((parseInt(event.target.style.top.replace('px', '')) + y) / 50).toString() +
                        Math.round((parseInt(event.target.style.left.replace('px', '')) + x) / 50).toString(),
                ];

                for (let i = 1; i < length; i++) {
                    if (length === 1) return;
                    if (direction === 'horizontal') {
                        newCoords.push(
                            Math.round((parseInt(event.target.style.top.replace('px', '')) + y) / 50).toString() +
                                Math.round(
                                    (parseInt(event.target.style.left.replace('px', '')) + x) / 50 + i
                                ).toString()
                        );
                    } else if (direction === 'vertical') {
                        newCoords.push(
                            Math.round((parseInt(event.target.style.top.replace('px', '')) + y) / 50 + i).toString() +
                                Math.round((parseInt(event.target.style.left.replace('px', '')) + x) / 50).toString()
                        );
                    }
                }
            })
            .on('dragend', function() {
                const shipID = eventTarget.getAttribute('id');
                const isVertical = eventTarget.getAttribute('data-direction') == 'vertical' ? true : false;

                const shipLength = Number(eventTarget.getAttribute('data-length'));
                const oldX = Number(newCoords[0][1]) - Math.round(x / 50);
                const oldY = Number(newCoords[0][0]) - Math.round(y / 50);
                const newX = Number(newCoords[0][1]);
                const newY = Number(newCoords[0][0]);

                if (shipLength === 1) {
                    that.player.gridMap[oldY][oldX] = '';
                } else if (!isVertical) {
                    for (let i = 0; i < shipLength; i++) {
                        that.player.gridMap[oldY][oldX + i] = '';
                    }
                } else if (isVertical) {
                    for (let i = 0; i < shipLength; i++) {
                        that.player.gridMap[oldY + i][oldX] = '';
                    }
                }

                that.createShip(newY, newX, that.shipFactory(shipLength), isVertical, that.player.gridMap, shipID);

                event.target.attributes[3].value = newY.toString() + newX.toString();

                eventTarget.style.webkitTransform = eventTarget.style.transform = 'translate(' + x + 'px, ' + y + 'px)';

                x = 0;
                y = 0;
            });
    },
    methods: {
        rotateShip(event) {
            if (this.gameFlow.gameStarted) return;
            const shipID = event.target.getAttribute('id');
            let isVertical = event.target.getAttribute('data-direction') == 'vertical' ? true : false;

            const shipLength = Number(event.target.getAttribute('data-length'));
            const coordX = parseInt(event.target.getAttribute('data-coords')[1]);
            const coordY = parseInt(event.target.getAttribute('data-coords')[0]);

            if (shipLength === 1) {
                return;
            } else if (!isVertical) {
                for (let i = 0; i < shipLength; i++) {
                    this.player.gridMap[coordY][coordX + i] = '';
                }
            } else if (isVertical) {
                for (let i = 0; i < shipLength; i++) {
                    this.player.gridMap[coordY + i][coordX] = '';
                }
            }

            isVertical = !isVertical;

            this.createShip(coordY, coordX, this.shipFactory(shipLength), isVertical, this.player.gridMap, shipID);

            [event.target.style.width, event.target.style.height] = [
                event.target.style.height,
                event.target.style.width,
            ];
        },
        playerAttack(ele) {
            if (this.gameFlow.playerTurn === true) {
                ele.target.style.fontSize = 20 + 'px';
                const data = JSON.parse(ele.target.dataset.coord);
                const cell = this.computer.gridMap[data.y][data.x];

                const index = cell[1];

                // empty square is ""
                if (cell == '') {
                    ele.target.textContent = '~';
                    this.gameFlow.playerTurn = false;
                    this.computerAttack();
                } else {
                    if (Number.isInteger(index)) {
                        cell[0].hit(index);
                        if (cell[0].isSunk()) {
                            this.compShipsAlive -= 1;

                            if (this.compShipsAlive == 0) {
                                this.gameFlow.gameStarted = false;
                                this.gameFlow.winner = true;
                                alert('You won!');
                            }

                            cell[0].coords.forEach((a) => {
                                let coordShip = JSON.stringify({ x: a[0], y: a[1] });
                                let shipSquare = document.querySelector(`[data-coord='${coordShip}']`);
                                shipSquare.style.color = 'red';

                                const possibleWater = [
                                    { x: a[0] - 1, y: a[1] },
                                    { x: a[0] + 1, y: a[1] },
                                    { x: a[0], y: a[1] - 1 },
                                    { x: a[0], y: a[1] + 1 },
                                ];
                                for (let i = 0; i < 4; i++) {
                                    const x = possibleWater[i].x;
                                    const y = possibleWater[i].y;
                                    if (this.isCoordValid({ x, y })) {
                                        // empty square is ""
                                        if (this.computer.gridMap[y][x] === '') {
                                            let coordSea = JSON.stringify(possibleWater[i]);
                                            let seaSquare = document.querySelector(`[data-coord='${coordSea}']`);
                                            seaSquare.textContent = '~';
                                            seaSquare.style.fontSize = 20 + 'px';
                                        }
                                    }
                                }
                            });
                        }
                    }
                    ele.target.textContent = 'X';
                }
            }
            return;
        },
        randomCoords() {
            const choseOption = () => {
                const index = Math.floor(Math.random() * this.compAttackOptions.length).toString();
                return this.compAttackOptions[index];
            };
            let attackCoords = choseOption();
            const x = parseInt(attackCoords[1]);
            const y = parseInt(attackCoords[0]);

            return { x, y };
        },

        computerAttack() {
            this.gameFlow.playerTurn = false;
            if (this.gameFlow.playerTurn) {
                return;
            }
            let coords = this.randomCoords();
            let x = coords.x;
            let y = coords.y;

            do {
                if (this.lastSuccesfullAttack.length == 1) {
                    x = this.lastSuccesfullAttack[0].x;
                    y = this.lastSuccesfullAttack[0].y;
                    if (this.compAttackOptions.includes(y.toString() + (x - 1).toString())) {
                        x = x - 1;
                    } else if (this.compAttackOptions.includes(y.toString() + (x + 1).toString())) {
                        x = x + 1;
                    } else if (this.compAttackOptions.includes((y - 1).toString() + x.toString())) {
                        y = y - 1;
                    } else if (this.compAttackOptions.includes((y + 1).toString() + x.toString())) {
                        y = y + 1;
                    }
                } else if (this.lastSuccesfullAttack.length >= 2) {
                    x = this.lastSuccesfullAttack[this.lastSuccesfullAttack.length - 1].x;
                    y = this.lastSuccesfullAttack[this.lastSuccesfullAttack.length - 1].y;
                    if (this.player.gridMap[y][x][0].isVertical) {
                        if (this.compAttackOptions.includes((y - 1).toString() + x.toString())) {
                            y = y - 1;
                        } else if (this.compAttackOptions.includes((y + 1).toString() + x.toString())) {
                            y = y + 1;
                        } else {
                            //algoritem gre proti prvemu delu ladje
                            y = y + this.lastSuccesfullAttack.length;
                        }
                    } else if (!this.player.gridMap[y][x][0].isVertical) {
                        if (this.compAttackOptions.includes(y.toString() + (x - 1).toString())) {
                            x = x - 1;
                        } else if (this.compAttackOptions.includes(y.toString() + (x + 1).toString())) {
                            x = x + 1;
                        } else {
                            x = x + this.lastSuccesfullAttack.length;
                        }
                    }
                }
                if (this.player.gridMap[y][x] === '') {
                    this.player.gridMap[y][x] = '~';

                    this.compAttackOptions.splice(this.compAttackOptions.indexOf(y.toString() + x.toString()), 1);
                    this.gameFlow.playerTurn = true;

                    const coordMiss = JSON.stringify({ x, y });
                    const seaSquare = document.querySelector(`[data-player-coord='${coordMiss}']`);

                    seaSquare.style.border = 'solid #c0392b 3px';
                    seaSquare.style.color = '#c0392b';
                    seaSquare.style.fontSize = 20 + 'px';
                    seaSquare.textContent = '~';

                    return;
                } else if (Array.isArray(this.player.gridMap[y][x])) {
                    this.compAttackOptions.splice(this.compAttackOptions.indexOf(y.toString() + x.toString()), 1);
                    this.player.gridMap[y][x][0].hit(this.player.gridMap[y][x][1]);
                    this.lastSuccesfullAttack.push({
                        x,
                        y,
                        part: this.player.gridMap[y][x][1],
                    });

                    let coordShip = JSON.stringify({ x, y });
                    let shipSquare = document.querySelector(`[data-player-coord='${coordShip}']`);

                    shipSquare.style.border = 'solid #c0392b 3px';
                    shipSquare.style.fontSize = 20 + 'px';
                    shipSquare.style.color = '#c0392b';
                    shipSquare.textContent = 'X';

                    if (this.player.gridMap[y][x][0].isSunk()) {
                        this.lastSuccesfullAttack = [];
                        this.player.gridMap[y][x][0].coords.forEach((a) => {
                            coordShip = JSON.stringify({ x: a[0], y: a[1] });
                            shipSquare = document.querySelector(`[data-player-coord='${coordShip}']`);

                            shipSquare.style.backgroundColor = '#f1c40f';

                            const possibleWater = [
                                { x: a[0] - 1, y: a[1] },
                                { x: a[0] + 1, y: a[1] },
                                { x: a[0], y: a[1] - 1 },
                                { x: a[0], y: a[1] + 1 },
                            ];

                            for (let i = 0; i < 4; i++) {
                                const wx = possibleWater[i].x;
                                const wy = possibleWater[i].y;
                                if (this.isCoordValid({ x: wx, y: wy })) {
                                    if (this.player.gridMap[wy][wx] === '') {
                                        let coordSea = JSON.stringify(possibleWater[i]);
                                        let seaSquare = document.querySelector(`[data-player-coord='${coordSea}']`);
                                        seaSquare.style.border = 'solid #c0392b 3px';
                                        seaSquare.style.color = '#c0392b';
                                        seaSquare.textContent = '~';
                                        seaSquare.style.fontSize = 20 + 'px';

                                        if (this.compAttackOptions.indexOf(wy.toString() + wx.toString()) !== -1)
                                            this.compAttackOptions.splice(
                                                this.compAttackOptions.indexOf(wy.toString() + wx.toString()),
                                                1
                                            );
                                    }
                                }
                            }
                        });
                        this.playerShipsAlive -= 1;
                    }
                }
                if (this.playerShipsAlive === 0) {
                    this.gameFlow.gameStarted = false;
                    this.gameFlow.winner = true;
                    alert('Computer won!');
                }
                coords = this.randomCoords();
                x = coords.x;
                y = coords.y;
            } while (!this.gameFlow.playerTurn);
        },
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
    box-shadow: rgb(180, 180, 255) 1px 0px 0px 0px, rgb(180, 180, 255) 0px 1px 0px 0px,
        rgb(180, 180, 255) 1px 1px 0px 0px, rgb(180, 180, 255) 1px 0px 0px 0px inset,
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

.active {
    border: solid 5px #2ecc71;
    overflow: hidden;
}
</style>
