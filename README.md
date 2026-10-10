import secrets
import string
import time

import aiosqlite
import discord


async def ensure_key_tables(db_name: str):
    """Create the redeem_keys table if it doesn't exist."""
    async with aiosqlite.connect(db_name) as db:
        await db.execute(
            """
            CREATE TABLE IF NOT EXISTS redeem_keys (
                key TEXT PRIMARY KEY,
                duration_seconds INTEGER NOT NULL,
                created_by INTEGER NOT NULL,
                created_at INTEGER NOT NULL,
                claimed_by INTEGER,
                claimed_at INTEGER,
                used INTEGER NOT NULL DEFAULT 0
            )
            """
        )
        await db.commit()


def generate_key(length: int = 12) -> str:
    """Generate a random uppercase alphanumeric key."""
    alphabet = string.ascii_uppercase + string.digits
    return "".join(secrets.choice(alphabet) for _ in range(length))


async def create_keys_for_duration(db_name: str, duration: str, amount: int, created_by: int, parse_duration):
    """Generate and store redemption keys for a duration."""
    seconds = parse_duration(duration)
    if not seconds:
        raise ValueError("Invalid duration")

    if amount < 1 or amount > 50:
        raise ValueError("Amount must be between 1 and 50")

    generated = []
    async with aiosqlite.connect(db_name) as db:
        for _ in range(amount):
            while True:
                new_key = generate_key()
                async with db.execute("SELECT 1 FROM redeem_keys WHERE key = ?", (new_key,)) as cursor:
                    exists = await cursor.fetchone()
                if not exists:
                    break

            await db.execute(
                """
                INSERT INTO redeem_keys (key, duration_seconds, created_by, created_at, claimed_by, claimed_at, used)
                VALUES (?, ?, ?, ?, NULL, NULL, 0)
                """,
                (new_key, seconds, created_by, int(time.time()))
            )
            generated.append(new_key)

        await db.commit()

    return generated


async def redeem_key_for_user(db_name: str, user_id: int, key: str, premium_role_id: int, guild):
    """Redeem a key and grant premium access to a user."""
    code = key.strip().upper()

    async with aiosqlite.connect(db_name) as db:
        async with db.execute(
            "SELECT duration_seconds, used FROM redeem_keys WHERE key = ?",
            (code,)
        ) as cursor:
            row = await cursor.fetchone()

        if not row:
            return {"success": False, "message": "Invalid key."}

        duration_seconds, used = row
        if used:
            return {"success": False, "message": "This key has already been used."}

        now = int(time.time())

        async with db.execute(
            "SELECT expires_at FROM users WHERE user_id = ?",
            (user_id,)
        ) as cursor:
            existing = await cursor.fetchone()

        current_expiry = existing[0] if existing else now
        new_expiry = max(current_expiry, now) + duration_seconds

        await db.execute(
            """
            INSERT INTO users (user_id, expires_at)
            VALUES (?, ?)
            ON CONFLICT(user_id)
            DO UPDATE SET expires_at = excluded.expires_at
            """,
            (user_id, new_expiry)
        )

        await db.execute(
            "UPDATE redeem_keys SET claimed_by = ?, claimed_at = ?, used = 1 WHERE key = ?",
            (user_id, now, code)
        )
        await db.commit()

    member = guild.get_member(user_id)
    role = guild.get_role(premium_role_id)

    if member and role and role not in member.roles:
        try:
            await member.add_roles(role)
        except Exception:
            pass

    return {
        "success": True,
        "message": f"Premium access activated for {duration_seconds // 86400} day(s).",
        "duration_seconds": duration_seconds,
    }


def register_key_commands(bot, tree, db_name, is_admin_check, make_embed, parse_duration, premium_role_id):
    """Register slash commands for key generation and redemption."""

    @tree.command(name="keygen")
    async def keygen(interaction: discord.Interaction, duration: str, amount: int = 1):
        if not is_admin_check(interaction.user):
            return await interaction.response.send_message(
                "You don't have permission to use this command.",
                ephemeral=True,
            )

        try:
            keys = await create_keys_for_duration(db_name, duration, amount, interaction.user.id, parse_duration)
        except ValueError as exc:
            return await interaction.response.send_message(str(exc), ephemeral=True)

        keys_text = "\n".join(f"`{key}`" for key in keys)

        await interaction.response.send_message(
            embed=make_embed(
                "Keys Generated",
                f"Generated {len(keys)} key(s) for `{duration}`:\n{keys_text}",
                0x00ff99,
                "keygen",
            )
        )

    @tree.command(name="redeem")
    async def redeem(interaction: discord.Interaction, key: str):
        if not interaction.guild:
            return await interaction.response.send_message(
                "This command can only be used in a server.",
                ephemeral=True,
            )

        result = await redeem_key_for_user(
            db_name,
            interaction.user.id,
            key,
            premium_role_id,
            interaction.guild,
        )

        if not result["success"]:
            return await interaction.response.send_message(
                embed=make_embed(
                    "Redemption Failed",
                    result["message"],
                    0xff0000,
                    "redeem",
                ),
                ephemeral=True,
            )

        await interaction.response.send_message(
            embed=make_embed(
                "Key Redeemed",
                f"{result['message']}\nYour premium access has been added to your account.",
                0x00ff99,
                "redeem",
            )
        )

    @tree.command(name="keys")
    async def keys_list(interaction: discord.Interaction):
        if not is_admin_check(interaction.user):
            return await interaction.response.send_message(
                "You don't have permission to use this command.",
                ephemeral=True,
            )

        async with aiosqlite.connect(db_name) as db:
            async with db.execute(
                "SELECT key, duration_seconds, used FROM redeem_keys ORDER BY created_at DESC LIMIT 20"
            ) as cursor:
                rows = await cursor.fetchall()

        if not rows:
            return await interaction.response.send_message("No keys have been generated yet.")

        lines = [
            f"`{row[0]}` | `{row[1]}s` | {'✅ used' if row[2] else '❌ unused'}"
            for row in rows
        ]

        await interaction.response.send_message(
            embed=make_embed(
                "Recent Keys",
                "\n".join(lines),
                0x00ff99,
                "keys",
            )
        )

    return {"keygen": keygen, "redeem": redeem, "keys": keys_list}


__all__ = [
    "ensure_key_tables",
    "generate_key",
    "create_keys_for_duration",
    "redeem_key_for_user",
    "register_key_commands",
]


