// CloudResourceAutoscaler.java
// Simulated Cloud Resource Autoscaler
// - Simulates multiple services producing CPU load
// - Autoscaler monitors metrics and scales instances up/down
// - Supports reactive and simple predictive (moving-average) scaling policies
//
// Compile & run: javac CloudResourceAutoscaler.java && java CloudResourceAutoscaler

import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.time.*;

public class CloudResourceAutoscaler {
    static class Service {
        final String name;
        final AtomicInteger instances = new AtomicInteger(1);
        final Random rnd = new Random();
        // per-instance CPU percent
        Service(String name) { this.name = name; }
        // simulate total CPU load as average utilization across instances (0..100)
        double sampleCpu() {
            // base load depends on number of instances and random variability
            double base = 20 + rnd.nextDouble() * 30; // baseline
            double burst = rnd.nextDouble() < 0.05 ? (30 + rnd.nextDouble() * 40) : 0;
            double cpuPerInstance = Math.min(100, base + burst);
            return cpuPerInstance;
        }
        void scaleTo(int n) { instances.set(Math.max(1, n)); }
        int getInstances() { return instances.get(); }
    }

    static class MetricsWindow {
        private final Deque<Double> window = new ArrayDeque<>();
        private final int maxSize;
        MetricsWindow(int maxSize) { this.maxSize = maxSize; }
        synchronized void add(double v) {
            window.addLast(v);
            if (window.size() > maxSize) window.removeFirst();
        }
        synchronized double avg() {
            if (window.isEmpty()) return 0;
            double s = 0;
            for (double d : window) s += d;
            return s / window.size();
        }
        synchronized int size() { return window.size(); }
    }

    static class Autoscaler {
        final Service svc;
        final MetricsWindow shortWindow = new MetricsWindow(5);  // recent
        final MetricsWindow longWindow = new MetricsWindow(30);  // trend
        final ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();
        final int minInstances;
        final int maxInstances;
        final double scaleUpThreshold;   // percent
        final double scaleDownThreshold; // percent
        final double scaleFactor; // multiplier
        Autoscaler(Service svc, int minInstances, int maxInstances,
                   double scaleUpThreshold, double scaleDownThreshold, double scaleFactor) {
            this.svc = svc;
            this.minInstances = minInstances;
            this.maxInstances = maxInstances;
            this.scaleUpThreshold = scaleUpThreshold;
            this.scaleDownThreshold = scaleDownThreshold;
            this.scaleFactor = scaleFactor;
        }
        void start() {
            executor.scheduleAtFixedRate(this::monitorAndScale, 1, 1, TimeUnit.SECONDS);
        }
        void stop() {
            executor.shutdownNow();
        }
        void ingest(double cpuSample) {
            shortWindow.add(cpuSample);
            longWindow.add(cpuSample);
        }
        void monitorAndScale() {
            try {
                double shortAvg = shortWindow.avg();
                double longAvg = longWindow.avg();
                int current = svc.getInstances();
                // predictive rule: if shortAvg significantly above longAvg -> scale up
                if (shortAvg > scaleUpThreshold && shortAvg > longAvg + 5) {
                    int target = Math.min(maxInstances, (int)Math.ceil(current * scaleFactor));
                    if (target > current) {
                        svc.scaleTo(target);
                        log("SCALE-UP", svc.name, current, target, shortAvg, longAvg);
                    }
                } else if (shortAvg < scaleDownThreshold && shortAvg < longAvg - 5) {
                    int target = Math.max(minInstances, (int)Math.floor(current / scaleFactor));
                    if (target < current) {
                        svc.scaleTo(target);
                        log("SCALE-DOWN", svc.name, current, target, shortAvg, longAvg);
                    }
                } else {
                    // conservative adjustments: if long-term trending up, pre-scale by 1
                    if (longAvg > scaleUpThreshold && current < maxInstances) {
                        svc.scaleTo(Math.min(maxInstances, current + 1));
                        log("PROACTIVE-SCALE-UP", svc.name, current, svc.getInstances(), shortAvg, longAvg);
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        void log(String action, String name, int before, int after, double s, double l) {
            System.out.printf("[%s] %s: %s instances %d -> %d (short=%.1f long=%.1f)%n",
                Instant.now(), action, name, before, after, s, l);
        }
    }

    public static void main(String[] args) throws Exception {
        Service web = new Service("web");
        Service worker = new Service("worker");
        Autoscaler webScaler = new Autoscaler(web, 1, 20, 65.0, 30.0, 1.5);
        Autoscaler workerScaler = new Autoscaler(worker, 1, 50, 70.0, 25.0, 1.8);
        webScaler.start();
        workerScaler.start();

        ScheduledExecutorService sim = Executors.newScheduledThreadPool(2);
        // feed metrics periodically
        sim.scheduleAtFixedRate(() -> {
            double cpu = web.sampleCpu() + Math.random()*10 - 5; // variability
            // CPU per instance normalized
            double perInstance = cpu;
            webScaler.ingest(perInstance);
            System.out.printf("[METRIC] web instances=%d cpu(approx) per-instance=%.1f%n", web.getInstances(), perInstance);
        }, 0, 1, TimeUnit.SECONDS);

        sim.scheduleAtFixedRate(() -> {
            double cpu = worker.sampleCpu() + Math.random()*20 - 10;
            workerScaler.ingest(cpu);
            System.out.printf("[METRIC] worker instances=%d cpu=%.1f%n", worker.getInstances(), cpu);
        }, 0, 1, TimeUnit.SECONDS);

        // run for a demo period
        Thread.sleep(30_000);
        sim.shutdownNow();
        webScaler.stop();
        workerScaler.stop();
        System.out.println("Demo complete.");
    }
}
