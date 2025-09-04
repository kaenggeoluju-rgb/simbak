# [심박 변이도 측정.html](https://github.com/user-attachments/files/22141601/default.html)
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<title>HRV 및 미분 그래프</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<style>
  body { font-family: Arial; text-align: center; margin: 20px; }
  canvas { width: 100%; max-width: 1200px; height: 500px; margin: 20px auto; display: block; }
  button { margin: 10px; padding: 10px 20px; font-size: 16px; cursor: pointer; }
</style>
</head>
<body>

<h2>HRV 및 미분 그래프</h2>

<input type="file" id="fileInput" accept=".csv">
<button id="sampleBtn">곡선 샘플 데이터 적용</button>

<canvas id="hrvChart"></canvas>
<canvas id="diffChart"></canvas>

<script>
let time = [];
let hrv = [];

// 수치 미분
function numericalDerivative(y, x){
    const dy = [];
    for(let i=0;i<y.length;i++){
        if(i===0){
            dy.push((y[i+1]-y[i])/(x[i+1]-x[i]));
        } else if(i===y.length-1){
            dy.push((y[i]-y[i-1])/(x[i]-x[i-1]));
        } else {
            dy.push((y[i+1]-y[i-1])/(x[i+1]-x[i-1]));
        }
    }
    return dy;
}

// CSV 읽기
document.getElementById('fileInput').addEventListener('change', function(e){
    const file = e.target.files[0];
    if(!file) return;
    const reader = new FileReader();
    reader.onload = function(event){
        const text = event.target.result;
        const data = parseCSV(text);
        time = data.time;
        hrv = data.hrv;
        drawCharts();
    };
    reader.readAsText(file);
});

// CSV 파싱
function parseCSV(text){
    const lines = text.trim().split('\n');
    const time = [];
    const hrv = [];
    for(let i=0;i<lines.length;i++){
        const [t,h] = lines[i].split(',').map(Number);
        time.push(t);
        hrv.push(h);
    }
    return {time, hrv};
}

// 샘플 버튼 (곡선 100개)
document.getElementById('sampleBtn').addEventListener('click', function(){
    time = [];
    hrv = [];
    for(let i=0;i<100;i++){
        time.push(i);
        hrv.push( +(0.8 + 0.02*Math.sin(0.2*i) + (Math.random()-0.5)*0.005).toFixed(3) );
    }
    drawCharts();
});

let hrvChart, diffChart;

function drawCharts(){
    const diff = numericalDerivative(hrv, time);

    // 음수 중 절댓값 최대값 인덱스
    let maxStressIdx = diff
        .map((v,i)=>v<0 ? {val:Math.abs(v), idx:i} : null)
        .filter(v=>v)
        .reduce((prev,curr)=> curr.val>prev.val ? curr : prev, {val:-1, idx:-1}).idx;

    if(hrvChart) hrvChart.destroy();
    if(diffChart) diffChart.destroy();

    // HRV 그래프
    hrvChart = new Chart(document.getElementById('hrvChart'), {
        type: 'line',
        data: {
            labels: time,
            datasets: [{
                label: 'HRV',
                data: hrv,
                borderColor: 'blue',
                borderWidth: 2,
                fill: false,
                tension: 0.3
            }]
        },
        options: {
            responsive: true,
            scales: {
                x: { title: { display: true, text: '시간 (초)' }, grid: { color: 'black' } },
                y: { title: { display: true, text: 'HRV (초)' }, grid: { color: 'black' } }
            }
        }
    });

    // 미분 그래프
    diffChart = new Chart(document.getElementById('diffChart'), {
        type: 'line',
        data: {
            labels: time,
            datasets: [{
                label: 'HRV 미분',
                data: diff,
                borderColor: 'red',
                borderWidth: 2,
                fill: false,
                tension: 0.3,
                pointBackgroundColor: function(context){
                    const idx = context.dataIndex;
                    return idx === maxStressIdx ? 'orange' : 'red';
                },
                pointRadius: 3,
                pointHoverRadius: 8
            }]
        },
        options: {
            responsive: true,
            scales: {
                x: { title: { display: true, text: '시간 (초)' }, grid: { color: 'black' } },
                y: {
                    title: { display: true, text: 'd(HRV)/dt' },
                    grid: {
                        color: function(context){
                            return context.tick.value === 0 ? 'black' : '#ccc';
                        },
                        lineWidth: function(context){
                            return context.tick.value === 0 ? 2 : 1;
                        }
                    }
                }
            },
            plugins: {
                tooltip: {
                    enabled: true,
                    callbacks: {
                        label: function(context) {
                            const yVal = context.raw;
                            const idx = context.dataIndex;
                            if(idx === maxStressIdx) return '스트레스 최대 급증 부분';
                            if(yVal > 0) return '스트레스 감소';
                            if(yVal < 0) return '스트레스 증가';
                            return '0';
                        }
                    }
                }
            }
        }
    });
}
</script>

</body>
</html>
