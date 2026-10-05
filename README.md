
import java.io.*;
import java.sql.*;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.time.format.DateTimeParseException;
import java.util.*;
import java.util.regex.Pattern;

// ---------------------------- DB Connection ----------------------------
class DBConnection1 {
    static Connection con;
    public static Connection getConnection() throws Exception {
        if (con == null) {
            Class.forName("com.mysql.cj.jdbc.Driver");
            con = DriverManager.getConnection("jdbc:mysql://localhost:3306/krish1", "root", "");
        }
        return con;
    }
}

// ---------------------------- Report Generator ----------------------------
class ReportGenerator {
    public static void generatePatientReport(int pid, String name, String symptoms, String prescription, String wardType, String doctorType) {
        try (FileWriter writer = new FileWriter("reports/PATIENT_" + pid + ".txt")) {
            String timestamp = getCurrentTimestamp();
            writer.write(" Report Date/Time: " + timestamp + "\n");
            writer.write(" Patient ID: " + pid + "\n");
            writer.write("Name: " + name + "\n");
            writer.write("Symptoms: " + symptoms + "\n");
            writer.write("Ward: " + wardType + "\n");
            writer.write("Assigned Doctor: " + doctorType + "\n");
            writer.write("Prescription: " + prescription + "\n");
            System.out.println(" Patient report saved.");
        } catch (IOException e) {
            System.out.println(" Error saving patient report.");
            e.printStackTrace();
        }
    }

    public static void updatePrescription(int pid, String prescription) {
        try (FileWriter writer = new FileWriter("reports/PATIENT_" + pid + ".txt", true)) {
            writer.write("Updated Prescription: " + prescription + "\n");
        } catch (IOException e) {
            System.out.println(" Error updating prescription.");
            e.printStackTrace();
        }
    }

    public static void generateBill(String name, int pid, int age, String gender, String symptoms, double room, double med, double lab, double extra, double finalTotal) {
        try (FileWriter writer = new FileWriter("bills/BILL_" + pid + ".txt")) {
            String timestamp = getCurrentTimestamp();
            writer.write("Patient Name: " + name + "\n");
            writer.write("Patient ID: " + pid + "\n");
            writer.write("Age " + age + "\n");
            writer.write("Gender " + gender + "\n");
            writer.write("Symptoms " + symptoms + "\n");
            writer.write(" Bill Date/Time: " + timestamp + "\n");
            writer.write(" BILL for Patient ID: " + pid + "\n");
            writer.write("Room Charge: ₹" + room + "\n");
            writer.write("Medicine Charge: ₹" + med + "\n");
            writer.write("Lab Charge: ₹" + lab + "\n");
            writer.write("Emergency Charge: ₹" + extra + "\n");
            writer.write("TOTAL: ₹" + finalTotal + "\n");
            System.out.println(" Bill saved.");
        } catch (IOException e) {
            System.out.println(" Error saving bill.");
            e.printStackTrace();
        }
    }

    private static String getCurrentTimestamp() {
        DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        return dtf.format(LocalDateTime.now());
    }
}

// ---------------------------- Lab ----------------------------
class Lab {
    public static final Map<String, Double> testRates = new HashMap<>();
    static {
        testRates.put("Blood Test", 400.0);
        testRates.put("X-Ray", 700.0);
        testRates.put("ECG", 500.0);
        testRates.put("MRI", 2500.0);
    }

    public static void showAllTests() {
        System.out.println(" Available Lab Tests:");
        for (String test : testRates.keySet()) {
            System.out.println(" - " + test + " : ₹" + testRates.get(test));
        }
    }
}

// ---------------------------- Patient ----------------------------
//INHERITANCE
class Patient1 extends user {
    static Scanner sc = new Scanner(System.in);

    public static void patientPanel(String username) throws Exception {
        System.out.println("\n⚕ Patient Panel: " + username);
        Connection con = null;
        try {
            con = DBConnection1.getConnection();
        } catch (Exception e) {
            System.out.println(" Database connection failed: " + e.getMessage());
            return;
        }

        System.out.print("Enter your name: ");
        String name = sc.nextLine();
        int age = -1;
        try {
            System.out.print("Enter age: ");
            age = sc.nextInt();
            sc.nextLine();
        } catch (InputMismatchException e) {
            System.out.println(" Invalid input for age. Please enter a number.");
            sc.nextLine();
            return;
        }

        System.out.print("Gender: ");
        String gender = sc.nextLine();
        System.out.print("Symptoms: ");
        String symptoms = sc.nextLine();

        // --- ADDED INSURANCE DETAILS ---
        System.out.print("Do you have health insurance? (yes/no): ");
        String insuranceAns = sc.nextLine().trim().toLowerCase();
        String insuranceNumber = null;
        if (insuranceAns.equals("yes")) {
            System.out.print("Enter 10-digit insurance number: ");
            String input = sc.nextLine().trim();
            if (input.length() == 10 && input.matches("\\d+")) {
                insuranceNumber = input;
            } else {
                System.out.println(" Invalid insurance number. It must be a 10-digit number. Proceeding without insurance.");
            }
        }
        // -------------------------------

        System.out.print("Do you want Emergency Admit? (yes/no): ");
        String emergencyAns = sc.nextLine().trim().toLowerCase();
        boolean isEmergency = false;
        String wardType = "General Ward";
        String doctorType = "Assistant Doctor";
        int priority = 2;

        if (emergencyAns.equals("yes")) {
            isEmergency = true;
            wardType = "Emergency Ward";
            doctorType = "Senior Doctor";
            priority = 1;
        } else {
            if (symptoms.toLowerCase().contains("chest pain") ||
                    symptoms.toLowerCase().contains("breath") ||
                    symptoms.toLowerCase().contains("unconscious") ||
                    symptoms.toLowerCase().contains("accident")) {
                isEmergency = true;
                wardType = "Emergency Ward";
                doctorType = "Senior Doctor";
                priority = 1;
            }
        }

        try {
            // Updated INSERT statement to include the insurance_number column
            PreparedStatement ps = con.prepareStatement(
                    "INSERT INTO patients (name, age, gender, symptoms, emergency, priority, username, discharged, insurance_number) VALUES (?, ?, ?, ?, ?, ?, ?, 0, ?)",
                    Statement.RETURN_GENERATED_KEYS
            );
            ps.setString(1, name);
            ps.setInt(2, age);
            ps.setString(3, gender);
            ps.setString(4, symptoms);
            ps.setInt(5, isEmergency ? 1 : 0);
            ps.setInt(6, priority);
            ps.setString(7, username);
            ps.setString(8, insuranceNumber); // Set the insurance number
            ps.executeUpdate();

            ResultSet keys = ps.getGeneratedKeys();
            if (keys.next()) {
                int pid = keys.getInt(1);
                ReportGenerator.generatePatientReport(pid, name, symptoms, "Not yet prescribed", wardType, doctorType);
            }
        } catch (SQLException e) {
            System.out.println(" Database error during patient admission: " + e.getMessage());
        }
    }
    public static boolean isValidDate(String dateString) {
        try {
            LocalDate date = LocalDate.parse(dateString, DateTimeFormatter.ofPattern("yyyy-MM-dd"));
            LocalDate today = LocalDate.now();
            return !date.isBefore(today);
        } catch (DateTimeParseException e) {
            return false;
        }
    }

    public static void bookAppointment(String username) {
        try {
            System.out.print("Enter preferred doctor username: ");
            String docUser = sc.nextLine();

            System.out.print("Enter appointment date (YYYY-MM-DD): ");
            String date = sc.nextLine();

            if (!isValidDate(date)) {
                System.out.println(" Invalid date. Please use YYYY-MM-DD format and ensure the date is not in the past.");
                return;
            }

            Connection con = DBConnection1.getConnection();
            PreparedStatement ps = con.prepareStatement(
                    "INSERT INTO appointments (patient_username, doctor_username, appointment_date, status) VALUES (?, ?, ?, ?)"
            );
            ps.setString(1, username);
            ps.setString(2, docUser);
            ps.setString(3, date);
            ps.setString(4, "Scheduled");

            ps.executeUpdate();
            System.out.println(" Appointment booked successfully.");

        } catch (SQLException e) {
            System.out.println(" Database error: " + e.getMessage());
        } catch (Exception e) {
            System.out.println(" An unexpected error occurred: " + e.getMessage());
        }
    }

}

// ---------------------------- Doctor ----------------------------
class Doctor1 extends user {
    static Scanner sc = new Scanner(System.in);

    public static void doctorPanel(String doctorUsername) throws Exception {
        Connection con = null;
        try {
            con = DBConnection1.getConnection();
        } catch (Exception e) {
            System.out.println(" Database connection failed: " + e.getMessage());
            return;
        }
        System.out.println("\n Doctor Panel: " + doctorUsername);

        try {
            PreparedStatement psList = con.prepareStatement(
                    "SELECT * FROM patients WHERE discharged=0 ORDER BY priority ASC, id ASC"
            );
            ResultSet rsList = psList.executeQuery();
            List<Integer> patientIds = new ArrayList<>();

            System.out.println(" Patient Queue (Emergency First):");
            boolean hasPatients = false;
            while (rsList.next()) {
                hasPatients = true;
                int pid = rsList.getInt("id");
                String pname = rsList.getString("name");
                String symptoms = rsList.getString("symptoms");
                int emergency = rsList.getInt("emergency");
                System.out.println(" - ID: " + pid + " | Name: " + pname + " | Symptoms: " + symptoms + " | " +
                        (emergency == 1 ? " Emergency" : "Normal"));
                patientIds.add(pid);
            }
            if (!hasPatients) {
                System.out.println("No patients in the queue.");
                return;
            }

            int pid = -1;
            try {
                System.out.print("Enter Patient ID to attend: ");
                pid = sc.nextInt();
                sc.nextLine();
            } catch (InputMismatchException e) {
                System.out.println(" Invalid input. Please enter a number.");
                sc.nextLine();
                return;
            }

            PreparedStatement infoPS = con.prepareStatement("SELECT * FROM patients WHERE id=?");
            infoPS.setInt(1, pid);
            ResultSet info = infoPS.executeQuery();
            if (info.next()) {
                System.out.println(" Name: " + info.getString("name"));
                System.out.println(" Symptoms: " + info.getString("symptoms"));
                System.out.println(" Age: " + info.getInt("age"));
                System.out.println(" Gender: " + info.getString("gender"));
            } else {
                System.out.println(" Patient not found.");
                return;
            }

            System.out.print("Enter prescription: ");
            String pres = sc.nextLine();

            PreparedStatement presPS = con.prepareStatement(
                    "INSERT INTO prescriptions (patient_id, doctor_username, prescription) VALUES (?, ?, ?)"
            );
            presPS.setInt(1, pid);
            presPS.setString(2, doctorUsername);
            presPS.setString(3, pres);
            presPS.executeUpdate();

            ReportGenerator.updatePrescription(pid, pres);
            System.out.println(" Prescription saved.");

            boolean assignLabTests = false;
            System.out.print("Do you want to assign lab tests? (yes/no): ");
            String ans = sc.nextLine();
            if (ans.equalsIgnoreCase("yes")) assignLabTests = true;

            while (assignLabTests) {
                Lab.showAllTests();
                System.out.print("Enter test name to assign (exact name): ");
                String test = sc.nextLine().trim();
                if (Lab.testRates.containsKey(test)) {
                    PreparedStatement testPS = con.prepareStatement(
                            "INSERT INTO lab_reports (patient_id, report_type, result) VALUES (?, ?, ?)"
                    );
                    testPS.setInt(1, pid);
                    testPS.setString(2, test);
                    testPS.setString(3, "Pending");
                    testPS.executeUpdate();
                    System.out.println(" Test assigned: " + test);
                } else {
                    System.out.println(" Invalid test name.");
                }
                System.out.print("Do you want to assign another lab test? (yes/no): ");
                String another = sc.nextLine().trim();
                assignLabTests = another.equalsIgnoreCase("yes");
            }

            System.out.print("View pending lab reports for attended patient? (yes/no): ");
            String viewLab = sc.nextLine();
            if (viewLab.equalsIgnoreCase("yes")) {
                viewLabReports(pid);
            }
        } catch (SQLException e) {
            System.out.println(" Database error: " + e.getMessage());
        }
    }

    public static void viewLabReports(int pid) throws Exception {
        Connection con = null;
        try {
            con = DBConnection1.getConnection();
        } catch (Exception e) {
            System.out.println(" Database connection failed: " + e.getMessage());
            return;
        }

        try {
            PreparedStatement ps = con.prepareStatement("SELECT * FROM lab_reports WHERE patient_id=? AND result='Pending'");
            ps.setInt(1, pid);
            ResultSet rs = ps.executeQuery();
            System.out.println("Pending Lab Reports:");
            boolean found = false;
            while (rs.next()) {
                found = true;
                System.out.println("Report: " + rs.getString("report_type") + ", Result: " + rs.getString("result"));
            }
            if (!found) {
                System.out.println("No pending lab reports.");
                return;
            }
            System.out.print("Mark a test as completed? (y/n): ");
            String ans = sc.nextLine();
            if (ans.equalsIgnoreCase("y")) {
                System.out.print("Enter test name: ");
                String test = sc.nextLine();
                PreparedStatement update = con.prepareStatement(
                        "UPDATE lab_reports SET result='Completed' WHERE patient_id=? AND report_type=?");
                update.setInt(1, pid);
                update.setString(2, test);
                update.executeUpdate();
                System.out.println(" Lab report updated.");
            }
        } catch (SQLException e) {
            System.out.println(" Database error: " + e.getMessage());
        }
    }
}

// ---------------------------- Billing ----------------------------
class Billing {
    static Scanner sc = new Scanner(System.in);

    public static void createBill(int patientId) throws Exception {
        Connection con = null;
        try {
            con = DBConnection1.getConnection();
        } catch (Exception e) {
            System.out.println(" Database connection failed: " + e.getMessage());
            return;
        }

        try {
            PreparedStatement patientPS = con.prepareStatement("SELECT * FROM patients WHERE id=?");
            patientPS.setInt(1, patientId);
            ResultSet patientRS = patientPS.executeQuery();
            if (!patientRS.next()) {
                System.out.println(" Patient not found.");
                return;
            }
            String name = patientRS.getString("name");
            int age = patientRS.getInt("age");
            String gender = patientRS.getString("gender");
            String symptoms = patientRS.getString("symptoms");
            String username = patientRS.getString("username");
            String insuranceNumber = patientRS.getString("insurance_number");
            boolean isEmergency = patientRS.getInt("emergency") == 1;
            double emergencyCharge = isEmergency ? 1500 : 0;

            double room = 0;
            int nights = 0;
            try {
                System.out.print("Enter number of nights lived in the hospital: ");
                nights = sc.nextInt();
                sc.nextLine();
                room = nights * 1000.0;
            } catch (InputMismatchException e) {
                System.out.println(" Invalid input for number of nights. Please enter a number.");
                sc.nextLine();
                return;
            }

            double medTotal = 0;
            PreparedStatement medPS = con.prepareStatement("SELECT * FROM user_medicines WHERE username=?");
            medPS.setString(1, username);
            ResultSet medRS = medPS.executeQuery();

            while (medRS.next()) {
                String medName = medRS.getString("medicine_name");
                int qty = medRS.getInt("quantity");
                double price = PharmacyStore.medicines.getOrDefault(medName, 0.0);
                double cost = price * qty;
                medTotal += cost;
            }

            double labTotal = 0;
            PreparedStatement labPS = con.prepareStatement("SELECT report_type FROM lab_reports WHERE patient_id=?");
            labPS.setInt(1, patientId);
            ResultSet labRS = labPS.executeQuery();

            while (labRS.next()) {
                String test = labRS.getString("report_type");
                double rate = Lab.testRates.getOrDefault(test, 0.0);
                labTotal += rate;
            }

            double total = room + medTotal + labTotal + emergencyCharge;

            // --- ADDED INSURANCE DISCOUNT LOGIC ---
            double finalTotal = total;
            if (insuranceNumber != null && insuranceNumber.length() == 10) {
                double discount = total * 0.15;
                finalTotal = total - discount;
                System.out.println(" Insurance found! Applying 15% discount.");
                System.out.printf("%-20s : ₹%.2f\n", "Discount (15%)", discount);
            }
            // ------------------------------------

            System.out.println("\n======= BILL FOR PATIENT: " + name + " (ID: " + patientId + ") =======");
            System.out.printf("%-20s : %s\n", "Name", name);
            System.out.printf("%-20s : %d\n", "Age", age);
            System.out.printf("%-20s : %s\n", "Gender", gender);
            System.out.printf("%-20s : %s\n", "Symptoms", symptoms);
            System.out.printf("%-20s : %s\n", "Emergency", isEmergency ? "Yes" : "No");
            System.out.println("---------------------------------------------");
            System.out.printf("%-20s : ₹%.2f\n", "Room Charge", room);
            System.out.printf("%-20s : ₹%.2f\n", "Medicine Charge", medTotal);
            System.out.printf("%-20s : ₹%.2f\n", "Lab Test Charge", labTotal);
            System.out.printf("%-20s : ₹%.2f\n", "Emergency Charge", emergencyCharge);
            if (insuranceNumber != null && insuranceNumber.length() == 10) {
                double discount = total * 0.15;
                System.out.printf("%-20s : ₹%.2f\n", "Insurance Discount", discount);
            }
            System.out.println("=============================================");
            System.out.printf("%-20s : ₹%.2f\n", "TOTAL", finalTotal);
            System.out.println("=============================================\n");
            PaymentMethod.processPayment(finalTotal);
            CallableStatement BillCS = con.prepareCall("{call insertBill(?,?,?,?,?,?) }");
            BillCS.setInt(1, patientId);
            BillCS.setDouble(2, room);
            BillCS.setDouble(3, medTotal);
            BillCS.setDouble(4, labTotal);
            BillCS.setDouble(5, emergencyCharge);
            BillCS.setDouble(6, finalTotal); // Pass the final discounted total
            BillCS.execute();

            ReportGenerator.generateBill(name, patientId, age, gender, symptoms, room, medTotal, labTotal, emergencyCharge, finalTotal);


        } catch (SQLException e) {
            System.out.println(" Database error: " + e.getMessage());
        }
    }
}

class PaymentMethod {
    static Scanner sc = new Scanner(System.in);

    public static void processPayment(double totalBill) {
        System.out.println("\n Select Payment Method:");
        System.out.println("1. Cash Payment");
        System.out.println("2. Card Payment");
        System.out.print("Enter your choice: ");
        int paymentChoice = -1;
        try {
            paymentChoice = sc.nextInt();
            sc.nextLine();
        } catch (InputMismatchException e) {
            System.out.println(" Invalid input. Please enter a number.");
            sc.nextLine();
            return;
        }

        switch (paymentChoice) {
            case 1:
                processCashPayment(totalBill);
                break;
            case 2:
                processCardPayment();
                break;
            default:
                System.out.println(" Invalid payment choice. Payment aborted.");
                break;
        }
    }

    private static void processCashPayment(double totalBill) {
        double cashReceived = 0;
        try {
            System.out.print("Enter cash amount received: ₹");
            cashReceived = sc.nextDouble();
            sc.nextLine();
        } catch (InputMismatchException e) {
            System.out.println(" Invalid amount. Please enter a number.");
            sc.nextLine();
            return;
        }

        double change = cashReceived - totalBill;
        if (change >= 0) {
            System.out.printf(" Payment done successfully! Your change is: ₹%.2f\n", change);
        } else {
            System.out.printf(" Insufficient cash. Amount needed: ₹%.2f\n", -change);
        }
    }

    private static void processCardPayment() {
        System.out.print("Enter 16-digit card number: ");
        String cardNumber = sc.nextLine();
        if (cardNumber.length() != 16 || !cardNumber.matches("\\d+")  ) {
            System.out.println(" Invalid card number. Must be 16 digits.");
            return;
        }
        if (!PaymentMethod.checkCard(cardNumber))
        {
            System.out.println(" Invalid card number. Please enter a valid card number.");
        }
        else

        System.out.print("Enter 3-digit CVV: ");
        String cvv = sc.nextLine();
        if (cvv.length() != 3 || !cvv.matches("\\d+")) {
            System.out.println(" Invalid CVV. Must be 3 digits.");
            return;
        }

        System.out.println(" Payment done successfully!");
    }
   static boolean checkCard(String cnum)
    {
        int a[]=new int[16];
        String number = cnum;
        int sum=0;
        for(int i=0;i<16;i++)
        {
            char c = number.charAt(i);
            int x;
            if(i%2!=0)
            {
                x = Character.getNumericValue(c);
            }
            else
            {
                x = Character.getNumericValue(c)*2;
            }
            if(x<=9)
            {
                a[i]=x;
            }
            else
            {
                x=x-9;
                a[i]=x;
            }
        }
        for(int i=0;i<a.length;i++)
        {
            sum=a[i]+sum;
        }
        if(sum%10==0)
        {
            System.out.println("Card Valid");
            return true;
        }
        else
        {
            System.out.println("Enter valid Card Number");
            return false;
        }
    }
}

// ---------------------------- Receptionist ----------------------------
class Receptionist {
    static Scanner sc = new Scanner(System.in);

    public static void receptionistPanel() throws Exception {
        System.out.println("\n Receptionist Panel");
        while (true) {
            System.out.println("1. Generate Bill");
            System.out.println("2. Search Patient");
            System.out.println("3. Discharge Patient");
            System.out.println("4. Back to Main Menu");
            System.out.print("Enter your choice: ");
            int choice = -1;
            try {
                choice = sc.nextInt();
                sc.nextLine();
            } catch (InputMismatchException e) {
                System.out.println(" Invalid input. Please enter a number.");
                sc.nextLine();
                continue;
            }
            switch (choice) {
                case 1:
                    System.out.print("Enter patient ID to generate bill: ");
                    int pid = -1;
                    try {
                        pid = sc.nextInt();
                        sc.nextLine();
                        Billing.createBill(pid);
                    } catch (InputMismatchException e) {
                        System.out.println(" Invalid patient ID. Please enter a number.");
                        sc.nextLine();
                    } catch (Exception e) {
                        System.out.println(" An error occurred: " + e.getMessage());
                    }
                    break;
                case 2:
                    try {
                        searchPatient();
                    } catch (SQLException e) {
                        System.out.println(" Database error: " + e.getMessage());
                    }
                    break;
                case 3:
                    System.out.print("Enter patient ID to Discharge Patient: ");
                    int patientId = -1;
                    try {
                        patientId = sc.nextInt();
                        sc.nextLine();
                        dischargePatient(patientId);
                    } catch (InputMismatchException e) {
                        System.out.println(" Invalid patient ID. Please enter a number.");
                        sc.nextLine();
                    } catch (SQLException e) {
                        System.out.println(" Database error: " + e.getMessage());
                    }
                    break;
                case 4:
                    return;
                default:
                    System.out.println(" Invalid choice");
            }
        }
    }

    public static void searchPatient() throws Exception {
        System.out.print("Enter patient name or ID: ");
        String query = sc.nextLine();
        Connection con = DBConnection1.getConnection();
        PreparedStatement ps;
        if (query.matches("\\d+")) {
            ps = con.prepareStatement("SELECT * FROM patients WHERE id=?");
            ps.setInt(1, Integer.parseInt(query));
        } else {
            ps = con.prepareStatement("SELECT * FROM patients WHERE name LIKE ?");
            ps.setString(1, "%" + query + "%");
        }
        ResultSet rs = ps.executeQuery();
        boolean found = false;
        while(rs.next()) {
            found = true;
            System.out.println("ID: " + rs.getInt("id") + ", Name: " + rs.getString("name") +
                    ", Age: " + rs.getInt("age") + ", Gender: " + rs.getString("gender") +
                    ", Symptoms: " + rs.getString("symptoms") +
                    ", Emergency: " + (rs.getInt("emergency")==1?"Yes":"No") +
                    ", Discharged: " + (rs.getInt("discharged")==1?"Yes":"No"));
        }
        if (!found) {
            System.out.println(" No patient found with that name or ID.");
        }
    }
    public static void dischargePatient(int patientId) throws Exception {

        Connection con = DBConnection1.getConnection();
        PreparedStatement psCheck = con.prepareStatement(
                "SELECT * FROM patients WHERE id=?");

        psCheck.setInt(1, patientId);
        ResultSet rsCheck = psCheck.executeQuery();

        if (!rsCheck.next()) {
            System.out.println(" Patient ID not found!");
            return;
        }

        if (rsCheck.getBoolean("discharged")) {
            System.out.println(" Patient is already discharged.");
            return;
        }

        PreparedStatement psReports = con.prepareStatement(
                "SELECT COUNT(*) FROM lab_reports WHERE patient_id=? AND result='Pending'"
        );
        psReports.setInt(1, patientId);
        ResultSet rsReports = psReports.executeQuery();

        if (rsReports.next() && rsReports.getInt(1) > 0) {
            System.out.println(" Patient cannot be discharged. Pending reports exist!");
            return;
        }
        PreparedStatement psUpdate = con.prepareStatement(
                "UPDATE patients SET discharged=1 WHERE id=?"
        );
        psUpdate.setInt(1, patientId);
        psUpdate.executeUpdate();

        System.out.println(" Patient discharged successfully!");
    }
}

// ---------------------------- Admin ----------------------------
class Admin extends user{
    public static void adminPanel() throws Exception {
        System.out.println("\n ADMIN PANEL");
        Connection con = null;
        try {
            con = DBConnection1.getConnection();
            Statement st = con.createStatement();
            ResultSet rs = st.executeQuery("SELECT * FROM users");
            System.out.println("Users:");
            while (rs.next()) {
                System.out.println(" - " + rs.getString("username") + " (" + rs.getString("role") + ")");
            }
        } catch (SQLException e) {
            System.out.println(" Database error: " + e.getMessage());
        } catch (Exception e) {
            System.out.println(" An unexpected error occurred: " + e.getMessage());
        }
    }
}

// ---------------------------- Pharmacy Store ----------------------------
class PharmacyStore {
    public static final Map<String, Double> medicines = new LinkedHashMap<>();
    static {
        medicines.put("Paracetamol", 20.0);
        medicines.put("Ibuprofen", 30.0);
        medicines.put("Amoxicillin", 50.0);
        medicines.put("Cetirizine", 15.0);
        medicines.put("Metformin", 40.0);
    }

    static Scanner sc = new Scanner(System.in);

    public static void showStore(String username) throws Exception {
        System.out.println("\n Pharmacy Store");

        int patientId = -1;
        try {
            System.out.print("Enter Patient ID for whom you want to add medicine details: ");
            patientId = sc.nextInt();
            sc.nextLine();
        } catch (InputMismatchException e) {
            System.out.println(" Invalid input. Please enter a number.");
            sc.nextLine();
            return;
        }

        Connection con = null;
        try {
            con = DBConnection1.getConnection();
            PreparedStatement checkPatient = con.prepareStatement("SELECT * FROM patients WHERE id=?");
            checkPatient.setInt(1, patientId);
            ResultSet rs = checkPatient.executeQuery();
            if (!rs.next()) {
                System.out.println(" Patient with ID " + patientId + " not found. Cannot add medicines.");
                return;
            }

            String patientUsername = rs.getString("username");

            System.out.println("Available Medicines:");
            int idx = 1;
            List<String> medList = new ArrayList<>(medicines.keySet());
            for (String med : medList) {
                System.out.printf("%d. %s - ₹%.2f\n", idx++, med, medicines.get(med));
            }

            System.out.print("Enter medicine numbers to purchase (comma separated, e.g., 1,3): ");
            String choices = sc.nextLine();
            String[] selectedIndexes = choices.split(",");

            for (String indexStr : selectedIndexes) {
                int medIdx;
                try {
                    medIdx = Integer.parseInt(indexStr.trim());
                } catch (NumberFormatException e) {
                    System.out.println(" Invalid number: " + indexStr);
                    continue;
                }
                if (medIdx < 1 || medIdx > medList.size()) {
                    System.out.println(" Invalid medicine selection: " + medIdx);
                    continue;
                }
                String medName = medList.get(medIdx - 1);
                int quantity;
                try {
                    System.out.print("Enter quantity for " + medName + ": ");
                    quantity = Integer.parseInt(sc.nextLine().trim());
                } catch (NumberFormatException e) {
                    System.out.println(" Invalid quantity.");
                    continue;
                }

                PreparedStatement ps = con.prepareStatement("INSERT INTO user_medicines (username, medicine_name, quantity) VALUES (?, ?, ?)");
                ps.setString(1, patientUsername);
                ps.setString(2, medName);
                ps.setInt(3, quantity);
                ps.executeUpdate();
                System.out.println(" Added " + quantity + " x " + medName + " to patient " + patientId + "'s record.");
            }
            System.out.println(" Purchase complete!");
        } catch (SQLException e) {
            System.out.println(" Database error: " + e.getMessage());
        } catch (Exception e) {
            System.out.println(" An unexpected error occurred: " + e.getMessage());
        }
    }
}

// ---------------------------- Main System ----------------------------
class HospitalManagementSystemFull {
    static Scanner sc = new Scanner(System.in);

    public static void main(String[] args) {
        new File("reports").mkdir();
        new File("bills").mkdir();

        while (true) {
            System.out.println("\n HOSPITAL MANAGEMENT SYSTEM");
            System.out.println("1. Add Patient");
            System.out.println("2. Attend Patient (Doctor)");
            System.out.println("3. Receptionist Panel");
            System.out.println("4. View Users (Admin)");
            System.out.println("5. Sign Up");
            System.out.println("6. Book Appointment");
            System.out.println("7. Pharmacy Store");
            System.out.println("8. Exit");
            System.out.print("Enter your choice: ");
            String input = sc.nextLine().trim();
            int mainChoice;
            try {
                mainChoice = Integer.parseInt(input);
            } catch (NumberFormatException e) {
                System.out.println(" Invalid input, please enter a number.");
                continue;
            }
            //polymorphism...same method,different forms.
            try {
                switch (mainChoice) {
                    case 1 :
                        loginAndExecute("patient", Patient1::patientPanel);
                        break;
                    case 2 :
                        loginAndExecute("doctor", Doctor1::doctorPanel);
                        break;
                    case 3 :
                        loginAndExecute("receptionist", (u) -> Receptionist.receptionistPanel());
                        break;
                    case 4 :
                        loginAndExecute("admin", (u) -> Admin.adminPanel());
                        break;
                    case 5 :
                        user.signup();
                        break;
                    case 6 :
                        loginAndExecute("patient", Patient1::bookAppointment);
                        break;
                    case 7 :
                        // Pass the role as a hint for the login screen. Any logged-in user can access the store.
                        loginAndExecute(null, PharmacyStore::showStore);
                        break;
                    case 8 :
                    {
                        System.out.println(" Exiting system.");
                        return;
                    }
                    default :
                        System.out.println(" Invalid choice");
                        break;
                }
            } catch (Exception e) {
                System.out.println(" An unexpected error occurred: " + e.getMessage());
            }
        }
    }

    @FunctionalInterface   //abstraction...show only essentials by hiding details.
    interface RoleExecutor {
        void execute(String username) throws Exception;
    }

    static void loginAndExecute(String expectedRole, RoleExecutor executor) throws Exception {
        System.out.println("\n LOGIN PANEL");
        System.out.print("Enter username: ");
        String username = sc.nextLine();
        System.out.print("Enter password: ");
        String password = sc.nextLine();

        Connection con = null;
        try {
            con = DBConnection1.getConnection();
            PreparedStatement ps = con.prepareStatement("SELECT * FROM users WHERE username=? AND password=?");
            ps.setString(1, username);
            ps.setString(2, password);
            ResultSet rs = ps.executeQuery();

            if (!rs.next()) {
                System.out.println(" Invalid credentials");
                return;
            }

            String role = rs.getString("role");
            if (expectedRole != null && !role.equals(expectedRole)) {
                System.out.println(" Access denied. You are logged in as a " + role);
                return;
            }

            System.out.println(" Login successful as " + role);
            executor.execute(username);
        } catch (SQLException e) {
            System.out.println(" Database error during login: " + e.getMessage());
        } catch (Exception e) {
            System.out.println(" An unexpected error occurred: " + e.getMessage());
        }
    }
}

class OTPService {

    private static int generatedOTP;
    public static int generateOTP() {
        Random rand = new Random();
        generatedOTP = 100000 + rand.nextInt(900000);
        System.out.println(" Your OTP is: " + generatedOTP);
        return generatedOTP;
    }

    public static boolean verifyOTP() {
        Scanner sc = new Scanner(System.in);
        System.out.print("Enter OTP: ");
        int enteredOTP = 0;
        try {
            enteredOTP = sc.nextInt();
        } catch (InputMismatchException e) {
            System.out.println(" Invalid input for OTP. Please enter a number.");
            sc.nextLine();
            return false;
        }

        if (enteredOTP == generatedOTP) {
            System.out.println(" OTP verified successfully!");
            return true;
        } else {
            System.out.println(" Invalid OTP. Please try again.");
            return false;
        }
    }
}

class user
{
    static Scanner sc = new Scanner(System.in);
    static void signup() throws Exception {
        String role = "";
        System.out.println("\n SIGNUP PANEL");
        System.out.println("Register as:");
        System.out.println("1. Patient \n2. Doctor \n3. Receptionist \n4. Admin");
        System.out.print("Enter your choice: ");
        int choiceRole = -1;
        try {
            choiceRole = sc.nextInt();
        } catch (InputMismatchException e) {
            System.out.println(" Invalid input. Please enter a number.");
            sc.nextLine();
            return;
        }

        switch (choiceRole) {
            case 1:
                role = "patient";
                break;
            case 2:
                role = "doctor";
                break;
            case 3:
                role = "receptionist";
                break;
            case 4:
                role = "admin";
                break;
            default:
                System.out.println(" Invalid role choice.");
                return;
        }
        sc.nextLine();

        System.out.print("Enter your Mobile Number: ");
        String mobileNumber = sc.nextLine();
        if (mobileNumber.length() != 10 || !mobileNumber.matches("\\d+")) {
            System.out.println(" Invalid mobile number. Must be a 10-digit number.");
            return;
        }

        OTPService.generateOTP();
        if (!OTPService.verifyOTP()) {
            System.out.println(" Registration failed. OTP did not match.");
            return;
        }

        System.out.print("Choose username: ");
        String username = sc.nextLine();
        System.out.print("Choose password: ");
        String password = sc.nextLine();

        Connection con = null;
        try {
            con = DBConnection1.getConnection();
            PreparedStatement checkUser = con.prepareStatement("SELECT * FROM users WHERE username=?");
            checkUser.setString(1, username);
            ResultSet rs = checkUser.executeQuery();
            if (rs.next()) {
                System.out.println(" Username already taken. Please choose another one.");
                return;
            }

            if (choiceRole == 2)
            {
                System.out.print("Enter Your Name: ");
                String D_name = sc.nextLine();
                System.out.print("Enter Your Email ID: ");
                String D_E_mail = sc.nextLine();
                System.out.print("Enter Your Gender: ");
                String D_gender = sc.nextLine();
                System.out.print("Enter Your specialization: ");
                String D_specialization = sc.nextLine();

                PreparedStatement D_ps = con.prepareStatement("INSERT INTO doctor (username, name, e_mail, gender, specialization) VALUES (?, ?, ?, ?, ?)");
                D_ps.setString(1, username);
                D_ps.setString(2, D_name);
                D_ps.setString(3, D_E_mail);
                D_ps.setString(4, D_gender);
                D_ps.setString(5, D_specialization);
                D_ps.executeUpdate();
            }

            PreparedStatement ps = con.prepareStatement("INSERT INTO users (username, password, role, Mobile_NO) VALUES (?, ?, ?, ?)");
            ps.setString(1, username);
            ps.setString(2, password);
            ps.setString(3, role);
            ps.setString(4, mobileNumber);
            ps.executeUpdate();
            System.out.println(" Registered successfully!");
        } catch (SQLException e) {
            System.out.println(" Database error during signup: " + e.getMessage());
        }
    }
}
