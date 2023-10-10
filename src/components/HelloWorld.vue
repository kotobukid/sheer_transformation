<template>
<div>
    <input type="file" @change="handleFileChange"/>
    <canvas ref="sourceCanvas" @click="handleClick" width="500" height="500"></canvas>
    <canvas ref="targetCanvas" :width="targetWidth" :height="targetHeight"></canvas>
</div>
</template>

<script>
import * as math from "mathjs";

function invertMatrix(matrix) {
    return math.inv(matrix);
}

function gaussJordan(A, b) {
    const n = b.length;
    // 拡大係数行列を作るにゃ
    for (let i = 0; i < n; i++) {
        A[i].push(b[i]);
    }

    // 前進消去（Forward Elimination）にゃ
    for (let i = 0; i < n; i++) {
        let maxRow = i;
        for (let k = i + 1; k < n; k++) {
            if (Math.abs(A[k][i]) > Math.abs(A[maxRow][i])) {
                maxRow = k;
            }
        }
        [A[i], A[maxRow]] = [A[maxRow], A[i]];

        for (let k = i + 1; k < n; k++) {
            const factor = A[k][i] / A[i][i];
            for (let j = i; j <= n; j++) {
                A[k][j] -= factor * A[i][j];
            }
        }
    }

    // 後退代入（Back Substitution）にゃ
    const x = new Array(n).fill(0);
    for (let i = n - 1; i >= 0; i--) {
        x[i] = A[i][n] / A[i][i];
        for (let k = i - 1; k >= 0; k--) {
            A[k][n] -= A[k][i] * x[i];
        }
    }
    return x;
}

function solve(A, b) {
    return gaussJordan(A, b);
}

function computeHomography(srcPoints, dstPoints) {
    const A = [];
    const b = [];

    for (let i = 0; i < 4; i++) {
        const [x, y] = [srcPoints[i].x, srcPoints[i].y];
        const [u, v] = [dstPoints[i].x, dstPoints[i].y];
        A.push([x, y, 1, 0, 0, 0, -u * x, -u * y]);
        A.push([0, 0, 0, x, y, 1, -v * x, -v * y]);
        b.push(u);
        b.push(v);
    }

    const h = solve(A, b);
    return [
        [h[0], h[1], h[2]],
        [h[3], h[4], h[5]],
        [h[6], h[7], 1]
    ];
}

function applyHomography(matrix, x, y) {
    const newX = (matrix[0][0] * x + matrix[0][1] * y + matrix[0][2]) / (matrix[2][0] * x + matrix[2][1] * y + matrix[2][2]);
    const newY = (matrix[1][0] * x + matrix[1][1] * y + matrix[1][2]) / (matrix[2][0] * x + matrix[2][1] * y + matrix[2][2]);
    return [newX, newY];
}

export default {
    data() {
        return {
            targetWidth: 500,
            targetHeight: 500,
            points: [],
            image: null,
        };
    },
    mounted() {
        this.loadImage();
    },
    methods: {
        loadImage() {
            this.image = new Image();
            this.image.src = 'your-image-url-here'; // 画像のURL
            this.image.onload = () => {
                this.drawImage(this.$refs.sourceCanvas, this.image);
            };
        },
        drawImage(canvas, image) {
            const ctx = canvas.getContext('2d');
            ctx.drawImage(image, 0, 0);
        },
        handleClick(event) {
            if (this.points.length < 4) {
                this.points.push({x: event.offsetX, y: event.offsetY});
            }
            if (this.points.length === 4) {
                this.points = this.sortPointsClockwise(this.points);
                this.drawRectangle(this.$refs.sourceCanvas, this.points)

                this.projectImage();
            }
        },
        pointInPolygon(x, y, polygon) {
            let inside = false;
            for (let i = 0, j = polygon.length - 1; i < polygon.length; j = i++) {
                const xi = polygon[i].x, yi = polygon[i].y;
                const xj = polygon[j].x, yj = polygon[j].y;

                const intersect = ((yi > y) !== (yj > y)) &&
                    (x < (xj - xi) * (y - yi) / (yj - yi) + xi);
                if (intersect) inside = !inside;
            }
            return inside;
        },
        drawRectangle(canvas, points) {
            if (points.length < 4) return;  // 4点未満では描画しないにゃ

            const ctx = canvas.getContext('2d');
            const [p1, p2, p3, p4] = this.sortPointsClockwise(points);

            ctx.beginPath();
            ctx.moveTo(p1.x, p1.y);
            ctx.lineTo(p2.x, p2.y);
            ctx.lineTo(p3.x, p3.y);
            ctx.lineTo(p4.x, p4.y);
            ctx.closePath();

            // ctx.fillStyle = 'rgba(128, 128, 128, 0.5)';  // グレーで半透明にゃ
            ctx.strokeStyle = 'rgba(255, 30, 30, 1)';  // 白い枠線
            // ctx.fill();
            ctx.stroke();
        },
        projectImage() {
            const sourceCtx = this.$refs.sourceCanvas.getContext('2d');
            const targetCtx = this.$refs.targetCanvas.getContext('2d');
            const imageData = sourceCtx.getImageData(0, 0, 500, 500);
            const newImageData = targetCtx.createImageData(this.targetWidth, this.targetHeight);

            const targetPoints = [
                {x: 0, y: 0},
                {x: this.targetWidth - 1, y: 0},
                {x: this.targetWidth - 1, y: this.targetHeight - 1},
                {x: 0, y: this.targetHeight - 1}
            ];

            const matrix = computeHomography(this.points, targetPoints);
            const inverseMatrix = invertMatrix(matrix);  // 逆行列を求めるにゃ

            for (let y = 0; y < 500; y++) {
                for (let x = 0; x < 500; x++) {
                    // if (!this.pointInPolygon(x, y, this.points)) {
                    //     continue;  // 四角形の外側はスキップにゃ
                    // }
                    // シアー変形を適用にゃ
                    const [newX, newY] = applyHomography(inverseMatrix, x, y);

                    if (this.pointInPolygon(newX, newY, this.points)) {
                        const index = (Math.floor(y) * this.targetHeight + Math.floor(x)) * 4;
                        const originalIndex = (Math.floor(newY) * this.targetWidth + Math.floor(newX)) * 4;
                        for (let i = 0; i < 4; i++) {
                            newImageData.data[index + i] = imageData.data[originalIndex + i];
                        }
                    }
                }
            }

            // 新しいCanvasに転写するにゃ
            targetCtx.putImageData(newImageData, 0, 0);
        },
        sortPointsClockwise(points) {
            const center = points.reduce(
                (acc, point) => ({x: acc.x + point.x / 4, y: acc.y + point.y / 4}),
                {x: 0, y: 0}
            );

            return points.slice().sort((a, b) => {
                const angleA = Math.atan2(a.y - center.y, a.x - center.x);
                const angleB = Math.atan2(b.y - center.y, b.x - center.x);
                return angleA - angleB;
            });
        },
        handleFileChange(event) {
            const file = event.target.files[0];
            if (file && file.type.match('image.*')) {
                const reader = new FileReader();
                reader.onload = (e) => {
                    this.image.src = e.target.result;
                    this.image.onload = () => {
                        this.drawImage(this.$refs.sourceCanvas, this.image);
                    };
                };
                reader.readAsDataURL(file);
                this.points = [];
            } else {
                console.error('Invalid file type. Please select an image file.');
            }
        }
    },
};
</script>

<style scoped lang='less'>
canvas {
    outline: 1px solid grey;
}
</style>
