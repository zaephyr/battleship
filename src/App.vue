<template>
  <div id="container">
    <app-the-game
      :gameFlow="gameFlow"
      :player="player"
      :computer="computer"
      :createShip="createShip"
      :shipFactory="shipFactory"
    ></app-the-game>
  </div>
</template>

<script>
import TheGame from "./components/TheGame";
export default {
  data() {
    return {
      gameFlow: {
        playerTurn: false,
        placingShips: true,
        gameStarted: false,
        playerWin: false,
        computerWin: false,
      },
      player: this.generateGrid(),
      computer: this.generateGrid(),
    };
  },
  components: {
    appTheGame: TheGame,
  },
  methods: {
    getRandomCoord() {
      return Math.floor(Math.random() * 10);
    },
    verticalOrNot() {
      return Math.random() <= 0.5;
    },
    shipFactory(len) {
      if (len >= 4) {
        len = 4;
      } else if (len <= 1) {
        len = 1;
      } else {
        len;
      }
      const hp = [];
      hp.length = len;
      hp.fill(0);

      const hit = (index) => {
        hp[index] = 1;
      };

      const isSunk = () => {
        const checkTrue = (part) => {
          return part == 1;
        };
        return hp.every(checkTrue);
      };
      return { hp, len, hit, isSunk };
    },
    isCoordValid({ x, y }) {
      const isCoordInRange = (coord) => 0 <= coord && coord <= 9;

      const validateCoord = (coord) => {
        if (!Number.isNaN(coord) && typeof coord === "number") {
          return isCoordInRange(coord);
        }
        return false;
      };
      return validateCoord(x) && validateCoord(y);
    },
    isShipPlaceable(start, end, board, isVertical) {
      if (isVertical) {
        // check center
        for (let i = start.y; i <= end.y; i++) {
          if (board[start.y][i] != "") {
            return false;
          }
        }

        // check left
        for (let i = start.y - 1; i <= end.y + 1; i++) {
          if (
            this.isCoordValid({ x: start.x - 1, y: i }) &&
            board[i][start.x - 1] != ""
          ) {
            return false;
          }
        }

        // check right
        for (let i = start.y - 1; i <= end.y + 1; i++) {
          if (
            this.isCoordValid({ x: start.x + 1, y: i }) &&
            board[i][start.x + 1] != ""
          ) {
            return false;
          }
        }

        //check top
        if (
          this.isCoordValid({ x: start.x, y: start.y - 1 }) &&
          board[start.y - 1][start.x] != ""
        ) {
          return false;
        }

        // check bottom
        if (
          this.isCoordValid({ x: start.x, y: end.y + 1 }) &&
          board[end.y + 1][start.x] != ""
        ) {
          return false;
        }

        // if all is good
        return true;
      }

      // horizontal position
      // check center
      for (let i = start.x; i <= end.x; i += 1) {
        if (board[start.y][i] != "") {
          return false;
        }
      }

      // check the top
      for (let i = start.x - 1; i <= end.x + 1; i += 1) {
        if (
          this.isCoordValid({ x: i, y: start.y - 1 }) &&
          board[start.y - 1][i] != ""
        ) {
          return false;
        }
      }

      // check the bottom
      for (let i = start.x + 1; i <= end.x + 1; i += 1) {
        if (
          this.isCoordValid({ x: i, y: start.y + 1 }) &&
          board[start.y + 1][i] != ""
        ) {
          return false;
        }
      }

      // check the left
      if (
        this.isCoordValid({ x: start.x - 1, y: start.y }) &&
        board[start.y][start.x - 1] != ""
      ) {
        return false;
      }

      // check the right
      if (
        this.isCoordValid({ x: start.x + 1, y: start.y }) &&
        board[start.y][start.x + 1] != ""
      ) {
        return false;
      }

      // if all is good
      return true;
    },
    isShipValid(ship, board, { x, y }, isVertical) {
      if (ship && this.isCoordValid({ x, y })) {
        const startPoint = { x, y };
        const endPoint = isVertical
          ? { x, y: y + ship.len - 1 }
          : { x: x + ship.len - 1, y };

        return (
          this.isCoordValid(endPoint) &&
          this.isShipPlaceable(startPoint, endPoint, board, isVertical)
        );
      }
      return false;
    },
    isShipReq(ship, boardInfo) {
      let reqTypeShip = {
        ship4: 1,
        ship3: 2,
        ship2: 3,
        ship1: 4,
      };
      const shipType = `ship${ship.len}`;
      return boardInfo[shipType] + 1 <= reqTypeShip[shipType];
    },
    createShip(y, x, ship, isVertical, board, id) {
      const newShip = ship;

      newShip.coords = [];
      newShip.isVertical = isVertical;

      for (let i = 0; i < ship.len; i++) {
        const funPossibleWater = (possibleWater) => {
          for (let j = 0; j < 4; j++) {
            const xx = possibleWater[j].x;
            const yy = possibleWater[j].y;
            console.log(xx, yy, this.isCoordValid({ xx, yy }));
            if (this.isCoordValid({ xx, yy })) {
              // empty square is ""
              if (board[yy][xx] == "") {
                board[yy][xx] = "~";
                console.log(xx, yy, "is valid");
              }
            }
          }
        };

        if (isVertical == false) {
          newShip.coords[i] = [x + i, y];
          board[y][x + i] = [newShip, i, id];
          const possibleWaterNotV = [
            { x: x + i - 1, y: y },
            { x: x + i + 1, y: y },
            { x: x + i, y: y - 1 },
            { x: x + i, y: y + 1 },
          ];

          funPossibleWater(possibleWaterNotV);
        } else if (isVertical == true) {
          board[y + i][x] = [newShip, i, id];
          newShip.coords[i] = [x, y + i];
          const possibleWaterIsV = [
            { x: x - 1, y: y + i },
            { x: x + 1, y: y + i },
            { x: x, y: y + i - 1 },
            { x: x, y: y + i + 1 },
          ];

          funPossibleWater(possibleWaterIsV);
        }
      }
    },
    generateShips(ship, board, boardInfo, { x, y }, isVertical, id) {
      if (this.isShipValid(ship, board, { x, y }, isVertical)) {
        if (this.isShipReq(ship, boardInfo)) {
          this.createShip(y, x, ship, isVertical, board, id);
          const shipType = `ship${ship.len}`;
          boardInfo[shipType] += 1;
        }
      }
    },

    generateGrid() {
      const gridMap = [];
      for (let x = 0; x < 10; x++) {
        gridMap.push([]);
        for (let y = 0; y < 10; y++) {
          gridMap[x].push("");
        }
      }

      const compShips = [4, 3, 3, 2, 2, 2, 1, 1, 1, 1];
      let boardInfo = {
        ship4: 0,
        ship3: 0,
        ship2: 0,
        ship1: 0,
      };
      let numOfLiveShips = 0;
      let ships = [];
      let shipID = 0;

      compShips.forEach((shipLen) => {
        let x = this.getRandomCoord();
        let y = this.getRandomCoord();

        let isVertical = this.verticalOrNot();

        while (
          !this.isShipValid(
            this.shipFactory(shipLen),
            gridMap,
            { x, y },
            isVertical
          )
        ) {
          x = this.getRandomCoord();
          y = this.getRandomCoord();
          isVertical = this.verticalOrNot();
        }

        if (
          this.isShipValid(
            this.shipFactory(shipLen),
            gridMap,
            { x, y },
            isVertical
          )
        ) {
          this.generateShips(
            this.shipFactory(shipLen),
            gridMap,
            boardInfo,
            { x, y },
            isVertical,
            shipID
          );

          const startPoint = { x, y };
          const endPoint = isVertical
            ? { x, y: y + shipLen - 1 }
            : { x: x + shipLen - 1, y };
          ships.push({
            length: shipLen,
            direction: isVertical ? "vertical" : "horizontal",
            position: [startPoint, endPoint],
          });
          shipID += 1;
          numOfLiveShips += 1;
        }
      });
      return { gridMap, ships, numOfLiveShips, boardInfo };
    },
  },
};
</script>
