import java.io.IOException;
import java.util.List;
import java.util.Map;
import java.util.Scanner;
import java.util.Set;
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;
import java.util.*;
import java.util.List;
import java.util.Map;
import java.util.Scanner;
import java.util.Set;



class Station {
    String name;
    Set<String> lines;

    public Station(String name) {
        this.name = name;
        this.lines = new HashSet<>();
    }

    public void addLine(String line) {
        lines.add(line);
    }

    @Override
    public String toString() {
        return name + " " + lines;
    }
}

class LineSegment {
    String line;
    Station from;
    Station to;
    double distance;

    public LineSegment(String line, Station from, Station to, double distance) {
        this.line = line;
        this.from = from;
        this.to = to;
        this.distance = distance;
    }

    @Override
    public String toString() {
        return line + ": " + from.name + " -> " + to.name + " (" + distance + " km)";
    }
}

class SubwaySystem {
    Map<String, Station> stations;
    List<LineSegment> lineSegments;

    public SubwaySystem(String filePath) throws IOException {
        stations = new HashMap<>();
        lineSegments = new ArrayList<>();
        loadSubwayData(filePath);
    }

    private void loadSubwayData(String filePath) throws IOException {
        BufferedReader reader = new BufferedReader(new FileReader(filePath));
        String line;
        String currentLine = "";

        while ((line = reader.readLine()) != null) {
            if (line.contains("号线站点间距")) {
                currentLine = line.split("号线")[0];
            } else if (line.contains("---")) {
                String[] parts = line.split("\t");
                String[] stationNames = parts[0].split("---");
                double distance = Double.parseDouble(parts[1]);

                addLineSegment(currentLine, stationNames[0], stationNames[1], distance);
            }
        }
        reader.close();
    }

    private void addLineSegment(String line, String from, String to, double distance) {
        Station fromStation = stations.computeIfAbsent(from, Station::new);
        Station toStation = stations.computeIfAbsent(to, Station::new);

        fromStation.addLine(line);
        toStation.addLine(line);

        lineSegments.add(new LineSegment(line, fromStation, toStation, distance));
        lineSegments.add(new LineSegment(line, toStation, fromStation, distance));
    }

    public Set<Station> getTransferStations() {
        Set<Station> transferStations = new HashSet<>();
        for (Station station : stations.values()) {
            if (station.lines.size() > 1) {
                transferStations.add(station);
            }
        }
        return transferStations;
    }

  public Map<Station, Double> getNearbyStations(String stationName, double maxDistance) {
        Station start = stations.get(stationName);
        if (start == null) {
            throw new IllegalArgumentException("站点不存在: " + stationName);
        }

        Map<Station, Double> nearbyStations = new HashMap<>();
        for (LineSegment segment : lineSegments) {
            if (segment.from.equals(start) && segment.distance <= maxDistance) {
                nearbyStations.put(segment.to, segment.distance);
            }
        }
        return nearbyStations;
    }

    public List<List<Station>> getAllPaths(String startName, String endName) {
        Station start = stations.get(startName);
        Station end = stations.get(endName);
        if (start == null || end == null) {
            throw new IllegalArgumentException("站点不存在: " + startName + " 或 " + endName);
        }
        List<List<Station>> paths = new ArrayList<>();
        findAllPaths(start, end, new HashSet<>(), new ArrayList<>(), paths);
        return paths;
    }

    private void findAllPaths(Station current, Station end, Set<Station> visited, List<Station> path, List<List<Station>> paths) {
        visited.add(current);
        path.add(current);

        if (current.equals(end)) {
            paths.add(new ArrayList<>(path));
        } else {
            for (LineSegment segment : lineSegments) {
                if (segment.from.equals(current) && !visited.contains(segment.to)) {
                    findAllPaths(segment.to, end, visited, path, paths);
                }
            }
        }

        path.remove(path.size() - 1);
        visited.remove(current);
    }

    public List<Station> getShortestPath(String startName, String endName) {
        Station start = stations.get(startName);
        Station end = stations.get(endName);
        if (start == null || end == null) {
            throw new IllegalArgumentException("站点不存在: " + startName + " 或 " + endName);
        }

        Map<Station, Double> distances = new HashMap<>();
        Map<Station, Station> previousStations = new HashMap<>();
        PriorityQueue<Station> queue = new PriorityQueue<>(Comparator.comparingDouble(distances::get));

        for (Station station : stations.values()) {
            if (station.equals(start)) {
                distances.put(station, 0.0);
            } else {
                distances.put(station, Double.MAX_VALUE);
            }
            queue.add(station);
        }

        while (!queue.isEmpty()) {
            Station current = queue.poll();

            for (LineSegment segment : lineSegments) {
                if (segment.from.equals(current)) {
                    Station neighbor = segment.to;
                    double newDist = distances.get(current) + segment.distance;
                    if (newDist < distances.get(neighbor)) {
                        queue.remove(neighbor);
                        distances.put(neighbor, newDist);
                        previousStations.put(neighbor, current);
                        queue.add(neighbor);
                    }
                }
            }
        }

        List<Station> path = new ArrayList<>();
        for (Station at = end; at != null; at = previousStations.get(at)) {
            path.add(at);
        }
        Collections.reverse(path);
        return path;
    }

    public void printPath(List<Station> path) {
        for (int i = 0; i < path.size() - 1; i++) {
            System.out.println(path.get(i).name + " -> " + path.get(i + 1).name);
        }
    }
   
