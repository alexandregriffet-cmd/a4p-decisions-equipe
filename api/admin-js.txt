// Fonction Vercel : /api/admin
// Réservée aux administrateurs. Crée des comptes, change un mot de passe, supprime un compte.
// Variables d'environnement à définir dans Vercel : SUPABASE_URL et SUPABASE_SERVICE_ROLE_KEY.
// La clé "service_role" donne tous les droits : elle ne doit JAMAIS figurer dans un fichier du dépôt.

const BASE = process.env.SUPABASE_URL;
const KEY = process.env.SUPABASE_SERVICE_ROLE_KEY;
const UUID = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
const EMAIL = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

async function call(path, opts = {}) {
  const r = await fetch(BASE + path, {
    ...opts,
    headers: { apikey: KEY, ...(KEY.startsWith("sb_") ? {} : { Authorization: "Bearer " + KEY }), "Content-Type": "application/json", ...(opts.headers || {}) },
  });
  const text = await r.text();
  let data = null;
  try { data = text ? JSON.parse(text) : null; } catch (e) { data = text; }
  return { ok: r.ok, status: r.status, data };
}

module.exports = async (req, res) => {
  const send = (code, obj) => res.status(code).json(obj);
  if (req.method !== "POST") return send(405, { error: "Méthode non autorisée" });
  if (!BASE || !KEY) return send(500, { error: "Variables SUPABASE_URL et SUPABASE_SERVICE_ROLE_KEY manquantes dans Vercel" });

  // 1. Qui appelle ?
  const token = (req.headers.authorization || "").replace(/^Bearer\s+/i, "");
  if (!token) return send(401, { error: "Connexion requise" });
  const who = await fetch(BASE + "/auth/v1/user", { headers: { apikey: KEY, Authorization: "Bearer " + token } });
  if (!who.ok) return send(401, { error: "Session invalide, reconnectez-vous" });
  const me = await who.json();

  // 2. Est-ce un administrateur actif ?
  const prof = await call("/rest/v1/profiles?id=eq." + me.id + "&select=role,active");
  if (!prof.ok || !prof.data || !prof.data[0] || prof.data[0].role !== "admin" || !prof.data[0].active) {
    return send(403, { error: "Réservé aux administrateurs" });
  }

  let body = req.body;
  if (typeof body === "string") { try { body = JSON.parse(body); } catch (e) { body = {}; } }
  body = body || {};

  // 3. Actions
  if (body.action === "create") {
    const email = String(body.email || "").trim().toLowerCase();
    const password = String(body.password || "");
    const name = String(body.display_name || "").trim().slice(0, 80);
    const role = body.role === "admin" ? "admin" : "coach";
    const quota = body.quota === null || body.quota === "" || body.quota === undefined ? null : Number(body.quota);
    if (!EMAIL.test(email)) return send(400, { error: "Adresse email invalide" });
    if (password.length < 8) return send(400, { error: "Mot de passe : 8 caractères minimum" });
    if (quota !== null && (!Number.isInteger(quota) || quota < 0)) return send(400, { error: "Nombre d'accès invalide" });
    let expires = body.expires_at ? new Date(body.expires_at) : null;
    if (expires && isNaN(expires)) return send(400, { error: "Date d'expiration invalide" });

    const created = await call("/auth/v1/admin/users", {
      method: "POST",
      body: JSON.stringify({ email, password, email_confirm: true, user_metadata: { display_name: name } }),
    });
    if (!created.ok) {
      const m = (created.data && (created.data.msg || created.data.message || created.data.error_description)) || "Création impossible";
      return send(400, { error: /already|registered|exists/i.test(m) ? "Cette adresse a déjà un compte" : m });
    }
    const id = created.data.id;
    // Le profil existe déjà (créé par le déclencheur) : on lui applique les accès voulus.
    const upd = await call("/rest/v1/profiles?id=eq." + id, {
      method: "PATCH",
      headers: { Prefer: "return=minimal" },
      body: JSON.stringify({ display_name: name || email.split("@")[0], role, quota, expires_at: expires ? expires.toISOString() : null, used: 0, active: true }),
    });
    if (!upd.ok) return send(500, { error: "Compte créé mais accès non appliqués : modifiez-les dans la liste." });
    return send(200, { ok: true, id });
  }

  if (body.action === "reset_password") {
    const password = String(body.password || "");
    if (!UUID.test(body.id || "")) return send(400, { error: "Identifiant invalide" });
    if (password.length < 8) return send(400, { error: "Mot de passe : 8 caractères minimum" });
    const r = await call("/auth/v1/admin/users/" + body.id, { method: "PUT", body: JSON.stringify({ password }) });
    return r.ok ? send(200, { ok: true }) : send(400, { error: "Changement impossible" });
  }

  if (body.action === "delete") {
    if (!UUID.test(body.id || "")) return send(400, { error: "Identifiant invalide" });
    if (body.id === me.id) return send(400, { error: "Vous ne pouvez pas supprimer votre propre compte" });
    const r = await call("/auth/v1/admin/users/" + body.id, { method: "DELETE" });
    return r.ok ? send(200, { ok: true }) : send(400, { error: "Suppression impossible" });
  }

  return send(400, { error: "Action inconnue" });
};
