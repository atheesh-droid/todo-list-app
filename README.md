// Student Placement Management System — backend API
// Express + in-memory data store (swap the store for a real DB later)

const express = require("express");
const cors = require("cors");

const app = express();
app.use(cors());
app.use(express.json());

// ---------- In-memory data ----------
let students = [
  { id: "s1", name: "Ananya Rao", roll: "CS21B045", branch: "Computer Science", cgpa: 8.9, skills: ["React", "SQL", "Python"] },
  { id: "s2", name: "Vikram Sethi", roll: "EC21B012", branch: "Electronics", cgpa: 7.6, skills: ["Embedded C", "VLSI"] },
  { id: "s3", name: "Meera Iyer", roll: "ME21B078", branch: "Mechanical", cgpa: 8.2, skills: ["AutoCAD", "SolidWorks"] },
  { id: "s4", name: "Rohan Das", roll: "CS21B091", branch: "Computer Science", cgpa: 9.3, skills: ["Java", "DSA", "AWS"] },
];

let companies = [
  { id: "c1", name: "Northbridge Analytics", role: "Data Analyst", ctc: 9.5, minCgpa: 7.0 },
  { id: "c2", name: "Ferro Systems", role: "Embedded Engineer", ctc: 7.2, minCgpa: 7.0 },
  { id: "c3", name: "Solstice Cloud", role: "SDE-1", ctc: 14.0, minCgpa: 8.0 },
];

let placements = [
  { id: "p1", studentId: "s1", companyId: "c1", stage: "Selected" },
  { id: "p2", studentId: "s4", companyId: "c3", stage: "Selected" },
  { id: "p3", studentId: "s2", companyId: "c2", stage: "Shortlisted" },
  { id: "p4", studentId: "s3", companyId: "c1", stage: "Applied" },
  { id: "p5", studentId: "s4", companyId: "c1", stage: "Rejected" },
];

const STAGES = ["Applied", "Shortlisted", "Selected", "Rejected"];
const uid = (prefix) => prefix + Math.random().toString(36).slice(2, 8);

// ---------- Helpers ----------
function notFound(res, what) {
  return res.status(404).json({ error: `${what} not found` });
}
function badRequest(res, message) {
  return res.status(400).json({ error: message });
}

// ================= Students =================
app.get("/api/students", (req, res) => {
  const { q } = req.query;
  if (!q) return res.json(students);
  const term = q.toLowerCase();
  res.json(students.filter((s) => s.name.toLowerCase().includes(term) || s.roll.toLowerCase().includes(term)));
});

app.get("/api/students/:id", (req, res) => {
  const student = students.find((s) => s.id === req.params.id);
  if (!student) return notFound(res, "Student");
  res.json(student);
});

app.post("/api/students", (req, res) => {
  const { name, roll, branch, cgpa, skills } = req.body;
  if (!name || !roll) return badRequest(res, "name and roll are required");
  const student = {
    id: uid("s"),
    name,
    roll,
    branch: branch || "",
    cgpa: Number(cgpa) || 0,
    skills: Array.isArray(skills) ? skills : (skills || "").split(",").map((x) => x.trim()).filter(Boolean),
  };
  students.push(student);
  res.status(201).json(student);
});

app.put("/api/students/:id", (req, res) => {
  const idx = students.findIndex((s) => s.id === req.params.id);
  if (idx === -1) return notFound(res, "Student");
  students[idx] = { ...students[idx], ...req.body, id: students[idx].id };
  res.json(students[idx]);
});

app.delete("/api/students/:id", (req, res) => {
  const exists = students.some((s) => s.id === req.params.id);
  if (!exists) return notFound(res, "Student");
  students = students.filter((s) => s.id !== req.params.id);
  placements = placements.filter((p) => p.studentId !== req.params.id);
  res.status(204).end();
});

// ================= Companies =================
app.get("/api/companies", (req, res) => res.json(companies));

app.get("/api/companies/:id", (req, res) => {
  const company = companies.find((c) => c.id === req.params.id);
  if (!company) return notFound(res, "Company");
  res.json(company);
});

app.post("/api/companies", (req, res) => {
  const { name, role, ctc, minCgpa } = req.body;
  if (!name || !role) return badRequest(res, "name and role are required");
  const company = { id: uid("c"), name, role, ctc: Number(ctc) || 0, minCgpa: Number(minCgpa) || 0 };
  companies.push(company);
  res.status(201).json(company);
});

app.put("/api/companies/:id", (req, res) => {
  const idx = companies.findIndex((c) => c.id === req.params.id);
  if (idx === -1) return notFound(res, "Company");
  companies[idx] = { ...companies[idx], ...req.body, id: companies[idx].id };
  res.json(companies[idx]);
});

app.delete("/api/companies/:id", (req, res) => {
  const exists = companies.some((c) => c.id === req.params.id);
  if (!exists) return notFound(res, "Company");
  companies = companies.filter((c) => c.id !== req.params.id);
  placements = placements.filter((p) => p.companyId !== req.params.id);
  res.status(204).end();
});

// ================= Placements =================
app.get("/api/placements", (req, res) => {
  const { studentId, companyId, stage } = req.query;
  let result = placements;
  if (studentId) result = result.filter((p) => p.studentId === studentId);
  if (companyId) result = result.filter((p) => p.companyId === companyId);
  if (stage) result = result.filter((p) => p.stage === stage);
  res.json(result);
});

app.post("/api/placements", (req, res) => {
  const { studentId, companyId, stage } = req.body;
  if (!studentId || !companyId) return badRequest(res, "studentId and companyId are required");
  if (!students.some((s) => s.id === studentId)) return badRequest(res, "unknown studentId");
  if (!companies.some((c) => c.id === companyId)) return badRequest(res, "unknown companyId");
  const finalStage = STAGES.includes(stage) ? stage : "Applied";
  const placement = { id: uid("p"), studentId, companyId, stage: finalStage };
  placements.push(placement);
  res.status(201).json(placement);
});

app.patch("/api/placements/:id/stage", (req, res) => {
  const { stage } = req.body;
  if (!STAGES.includes(stage)) return badRequest(res, `stage must be one of ${STAGES.join(", ")}`);
  const idx = placements.findIndex((p) => p.id === req.params.id);
  if (idx === -1) return notFound(res, "Placement");
  placements[idx].stage = stage;
  res.json(placements[idx]);
});

app.delete("/api/placements/:id", (req, res) => {
  const exists = placements.some((p) => p.id === req.params.id);
  if (!exists) return notFound(res, "Placement");
  placements = placements.filter((p) => p.id !== req.params.id);
  res.status(204).end();
});

// ================= Dashboard summary =================
app.get("/api/dashboard", (req, res) => {
  const placedIds = new Set(placements.filter((p) => p.stage === "Selected").map((p) => p.studentId));
  const placedCount = placedIds.size;
  const placementRate = students.length ? Math.round((placedCount / students.length) * 100) : 0;

  const selected = placements.filter((p) => p.stage === "Selected");
  const avgCtc = selected.length
    ? (selected.reduce((sum, p) => sum + (companies.find((c) => c.id === p.companyId)?.ctc || 0), 0) / selected.length).toFixed(1)
    : "0.0";

  const stageCounts = STAGES.map((stage) => ({
    stage,
    count: placements.filter((p) => p.stage === stage).length,
  }));

  const topCompany = [...companies].sort((a, b) => b.ctc - a.ctc)[0] || null;

  res.json({
    totalStudents: students.length,
    placedCount,
    placementRate,
    avgCtc: Number(avgCtc),
    stageCounts,
    topCompany,
  });
});

app.get("/api/health", (req, res) => res.json({ status: "ok" }));

const PORT = process.env.PORT || 4000;
app.listen(PORT, () => console.log(`Placement system API running on port ${PORT}`));

module.exports = app;
