![[司书博-U202211648-OOP大作业.pdf](https://github.com/sishubo/zuoye1/files/15383850/-U202211648-OOP.pdf)

image](https://github.com/sishubo/zuoye1/assets/168086953/54f3cc7f-dc03-4c50-8612-ca6702b99806)

	简要说明：为用户提供一些查询功能。
	主事件流：用户输入某一站点名称，系统查询后告诉用户该站点拥有的线路数；若输入某条线路名称，则系统查询该线路拥有的站点数；在用户输入起点和终点之后，系统自动为其提供两站之间的最优路线，以及对应的换乘次数、公里数和票价等。
	其他事件流：无。
	前提事件：用户需要进行站点和线路信息查询。
	事后事件：无。


![image](https://github.com/sishubo/zuoye1/assets/168086953/ed177482-1a4a-49df-83b7-e69234a859ce)





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

class LineSegment {// LineSegment类表示地铁线路上的一段
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

class SubwaySystem {// SubwaySystem类表示整个地铁系统
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

    public Set<Station> getTransferStations() {// 获取所有中转站的方法
        Set<Station> transferStations = new HashSet<>();
        for (Station station : stations.values()) {
            if (station.lines.size() > 1) {
                transferStations.add(station);
            }
        }
        return transferStations;
    }

  public Map<Station, Double> getNearbyStations(String stationName, double maxDistance) {// 根据站点名称和最大距离获取附近站点的方法
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
   
public double calculateFare(List<Station> path) {
        double totalDistance = 0;
        for (int i = 0; i < path.size() - 1; i++) {
            for (LineSegment segment : lineSegments) {
                if (segment.from.equals(path.get(i)) && segment.to.equals(path.get(i + 1))) {
                    totalDistance += segment.distance;
                    break;
                }
            }
        }
        return totalDistance;
    }

    public double calculateFareWithDiscount(List<Station> path, String ticketType) {
        double baseFare = calculateFare(path);
        switch (ticketType) {
            case "武汉通":
                return baseFare * 0.9;
            case "日票":
                return 0;
            default:
                return baseFare;
        }
    }
}

public class TestCLI {
    public static void main(String[] args) {
        try {
            SubwaySystem subwaySystem = new SubwaySystem("F:\\桌面\\subway.txt1");
            Scanner scanner = new Scanner(System.in);
            while (true) {
                System.out.println("请选择操作:");
                System.out.println("1. 识别地铁中转站");
                System.out.println("2. 查找附近站点");
                System.out.println("3. 查找所有路径");
                System.out.println("4. 查找最短路径");
                System.out.println("5. 计算乘车费用");
                System.out.println("6. 计算不同票种的票价");
                System.out.println("7. 退出");

                int choice = scanner.nextInt();
                scanner.nextLine(); // consume newline

                switch (choice) {
                    case 1:
                        Set<Station> transferStations = subwaySystem.getTransferStations();
                        System.out.println("Transfer Stations: " + transferStations);
                        break;

                    case 2:
                        System.out.println("输入站点名称:");
                        String stationName = scanner.nextLine();
                        System.out.println("输入距离:");
                        double distance = scanner.nextDouble();
                        scanner.nextLine(); // consume newline

                        try {
                            Map<Station, Double> nearbyStations = subwaySystem.getNearbyStations(stationName, distance);
                            System.out.println("Nearby Stations: " + nearbyStations);
                        } catch (IllegalArgumentException e) {
                            System.out.println(e.getMessage());
                        }
                        break;

                    case 3:
                        System.out.println("输入起点站:");
                        String startStation = scanner.nextLine();
                        System.out.println("输入终点站:");
                        String endStation = scanner.nextLine();

                        try {
                            List<List<Station>> allPaths = subwaySystem.getAllPaths(startStation, endStation);
                            System.out.println("All Paths: " + allPaths);
                        } catch (IllegalArgumentException e) {
                            System.out.println(e.getMessage());
                        }
                        break;

                    case 4:
                        System.out.println("输入起点站:");
                        startStation = scanner.nextLine();
                        System.out.println("输入终点站:");
                        endStation = scanner.nextLine();

                        try {
                            List<Station> shortestPath = subwaySystem.getShortestPath(startStation, endStation);
                            System.out.println("Shortest Path: " + shortestPath);
                            subwaySystem.printPath(shortestPath);
                        } catch (IllegalArgumentException e) {
                            System.out.println(e.getMessage());
                        }
                        break;

                    case 5:
                        System.out.println("输入起点站:");
                        startStation = scanner.nextLine();
                        System.out.println("输入终点站:");
                        endStation = scanner.nextLine();

                        try {
                            List<Station> path = subwaySystem.getShortestPath(startStation, endStation);
                            double fare = subwaySystem.calculateFare(path);
                            System.out.println("Fare: " + fare);
                        } catch (IllegalArgumentException e) {
                            System.out.println(e.getMessage());
                        }
                        break;

                    case 6:
                        System.out.println("输入起点站:");
                        startStation = scanner.nextLine();
                        System.out.println("输入终点站:");
                        endStation = scanner.nextLine();
                        System.out.println("输入票种 (普通, 武汉通, 日票):");
                        String ticketType = scanner.nextLine();

                        try {
                            List<Station> path = subwaySystem.getShortestPath(startStation, endStation);
                            double discountedFare = subwaySystem.calculateFareWithDiscount(path, ticketType);
                            System.out.println("Discounted Fare (" + ticketType + "): " + discountedFare);
                        } catch (IllegalArgumentException e) {
                            System.out.println(e.getMessage());
                        }
                        break;

                    case 7:
                        scanner.close();
                        return;

                    default:
                        System.out.println("无效选择，请重新选择。");
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

//选做题
import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.io.IOException;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class SubwaySystemGUI {
    private SubwaySystem subwaySystem;

    public SubwaySystemGUI(String filePath) throws IOException {
        subwaySystem = new SubwaySystem(filePath);
        createAndShowGUI();
    }

    private void createAndShowGUI() {
        JFrame frame = new JFrame("Subway System");
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setSize(600, 400);

        JPanel panel = new JPanel();
        panel.setLayout(new GridLayout(8, 1));

        JButton transferStationsButton = new JButton("识别地铁中转站");
        transferStationsButton.addActionListener(e -> showTransferStations());
        panel.add(transferStationsButton);

        JButton nearbyStationsButton = new JButton("查找附近站点");
        nearbyStationsButton.addActionListener(e -> showNearbyStations());
        panel.add(nearbyStationsButton);

        JButton allPathsButton = new JButton("查找所有路径");
        allPathsButton.addActionListener(e -> showAllPaths());
        panel.add(allPathsButton);

        JButton shortestPathButton = new JButton("查找最短路径");
        shortestPathButton.addActionListener(e -> showShortestPath());
        panel.add(shortestPathButton);

        JButton calculateFareButton = new JButton("计算乘车费用");
        calculateFareButton.addActionListener(e -> calculateFare());
        panel.add(calculateFareButton);

        JButton discountedFareButton = new JButton("计算不同票种的票价");
        discountedFareButton.addActionListener(e -> calculateDiscountedFare());
        panel.add(discountedFareButton);

        JButton exitButton = new JButton("退出");
        exitButton.addActionListener(e -> System.exit(0));
        panel.add(exitButton);

        frame.getContentPane().add(panel);
        frame.setVisible(true);
    }

    private void showTransferStations() {
        Set<Station> transferStations = subwaySystem.getTransferStations();
        JOptionPane.showMessageDialog(null, "Transfer Stations: " + transferStations);
    }

    private void showNearbyStations() {
        String stationName = JOptionPane.showInputDialog("输入站点名称:");
        String distanceStr = JOptionPane.showInputDialog("输入距离:");
        double distance = Double.parseDouble(distanceStr);

        try {
            Map<Station, Double> nearbyStations = subwaySystem.getNearbyStations(stationName, distance);
            JOptionPane.showMessageDialog(null, "Nearby Stations: " + nearbyStations);
        } catch (IllegalArgumentException e) {
            JOptionPane.showMessageDialog(null, e.getMessage());
        }
    }

    private void showAllPaths() {
        String startStation = JOptionPane.showInputDialog("输入起点站:");
        String endStation = JOptionPane.showInputDialog("输入终点站:");

        try {
            List<List<Station>> allPaths = subwaySystem.getAllPaths(startStation, endStation);
            JOptionPane.showMessageDialog(null, "All Paths: " + allPaths);
        } catch (IllegalArgumentException e) {
            JOptionPane.showMessageDialog(null, e.getMessage());
        }
    }

    private void showShortestPath() {
        String startStation = JOptionPane.showInputDialog("输入起点站:");
        String endStation = JOptionPane.showInputDialog("输入终点站:");

        try {
            List<Station> shortestPath = subwaySystem.getShortestPath(startStation, endStation);
            JOptionPane.showMessageDialog(null, "Shortest Path: " + shortestPath);
            subwaySystem.printPath(shortestPath);
        } catch (IllegalArgumentException e) {
            JOptionPane.showMessageDialog(null, e.getMessage());
        }
    }

    private void calculateFare() {
        String startStation = JOptionPane.showInputDialog("输入起点站:");
        String endStation = JOptionPane.showInputDialog("输入终点站:");

        try {
            List<Station> path = subwaySystem.getShortestPath(startStation, endStation);
            double fare = subwaySystem.calculateFare(path);
            JOptionPane.showMessageDialog(null, "Fare: " + fare);
        } catch (IllegalArgumentException e) {
            JOptionPane.showMessageDialog(null, e.getMessage());
        }
    }

    private void calculateDiscountedFare() {
        String startStation = JOptionPane.showInputDialog("输入起点站:");
        String endStation = JOptionPane.showInputDialog("输入终点站:");
        String ticketType = JOptionPane.showInputDialog("输入票种 (普通, 武汉通, 日票):");

        try {
            List<Station> path = subwaySystem.getShortestPath(startStation, endStation);
            double discountedFare = subwaySystem.calculateFareWithDiscount(path, ticketType);
            JOptionPane.showMessageDialog(null, "Discounted Fare (" + ticketType + "): " + discountedFare);
        } catch (IllegalArgumentException e) {
            JOptionPane.showMessageDialog(null, e.getMessage());
        }
    }

    public static void main(String[] args) {
        try {
            new SubwaySystemGUI("F:\\桌面\\subway.txt1");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
