<template>
  <div class="admin-dashboard">
    <h1 class="welcome-message">🔐 Editar Permisos</h1>

    <div class="admin-info">
      <!-- Selección de Usuario -->
      <h2>👤 Seleccionar Usuario</h2>
      <div class="form-group">
        <select v-model="usuarioSeleccionado" @change="asignarPermisosUsuario">
          <option disabled value="">Seleccione un usuario...</option>
          <option v-for="usuario in usuarios" :key="usuario.idUsuario" :value="usuario.idUsuario">
            {{ usuario.Nombre_usuario }} ({{ usuario.Persona?.Correo }})
          </option>
        </select>
      </div>

      <!-- Tabla de Permisos -->
      <h2>📋 Permisos por Módulo</h2>
      <table class="permissions-table">
        <thead>
          <tr>
            <th>Módulo</th>
            <th>Crear</th>
            <th>Leer</th>
            <th>Actualizar</th>
            <th>Eliminar</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="modulo in modulos" :key="modulo.idModulo">
            <td>{{ modulo.Nombre }}</td>
            <td>
              <select v-model="permisos[modulo.idModulo].crear">
                <option :value="true">✅ Sí</option>
                <option :value="false">❌ No</option>
              </select>
            </td>
            <td>
              <select v-model="permisos[modulo.idModulo].leer">
                <option :value="true">✅ Sí</option>
                <option :value="false">❌ No</option>
              </select>
            </td>
            <td>
              <select v-model="permisos[modulo.idModulo].actualizar">
                <option :value="true">✅ Sí</option>
                <option :value="false">❌ No</option>
              </select>
            </td>
            <td>
              <select v-model="permisos[modulo.idModulo].eliminar">
                <option :value="true">✅ Sí</option>
                <option :value="false">❌ No</option>
              </select>
            </td>
          </tr>
        </tbody>
      </table>

      <!-- Botón para Guardar -->
      <button @click="confirmarGuardado" class="save-button" :disabled="!usuarioSeleccionado">
        ✅ Actualizar Permisos
      </button>

      <p v-if="mensajeError" class="error-message">{{ mensajeError }}</p>
      <p v-if="mensajeExito" class="success-message">{{ mensajeExito }}</p>
    </div>
  </div>
</template>

<script>
import axios from "axios";

export default {
  data() {
    return {
      usuarios: [],
      modulos: [],
      permisos: {},
      permisosTodos: [], // Aquí guardamos todos los permisos de la API
      usuarioSeleccionado: "",
      permisosIniciales: {},
      mensajeError: "",
      mensajeExito: "",
    };
  },
  methods: {
    // 📥 Cargar todos los datos (usuarios, módulos y permisos)
    async cargarDatos() {
      try {
        const [usuariosResponse, modulosResponse, permisosResponse] = await Promise.all([
          axios.get("http://localhost:3001/api/usuario/all", { withCredentials: true }),
          axios.get("http://localhost:3001/api/modulo/all", { withCredentials: true }),
          axios.get("http://localhost:3001/api/permiso/all", { withCredentials: true }) // 🔥 Traemos todos los permisos
        ]);

        this.usuarios = usuariosResponse.data;
        this.modulos = modulosResponse.data;
        this.permisosTodos = permisosResponse.data;

        // Inicializar permisos vacíos para cada módulo
        this.permisos = this.modulos.reduce((acc, modulo) => {
          acc[modulo.idModulo] = { crear: false, leer: false, actualizar: false, eliminar: false };
          return acc;
        }, {});
      } catch (error) {
        console.error("❌ Error cargando datos:", error);
      }
    },

    // 📥 Asignar permisos al usuario seleccionado
    asignarPermisosUsuario() {
      if (!this.usuarioSeleccionado) return;

      // Filtrar los permisos que corresponden al usuario seleccionado
      const permisosUsuario = this.permisosTodos.filter(p => p.idUsuarioRol === this.usuarioSeleccionado);

      console.log("🔍 Permisos cargados para el usuario:", permisosUsuario);

      // Resetear y asignar permisos existentes
      this.permisos = this.modulos.reduce((acc, modulo) => {
        const permiso = permisosUsuario.find((p) => p.idModulo === modulo.idModulo);
        acc[modulo.idModulo] = permiso
          ? {
              crear: permiso.Crear,
              leer: permiso.Leer,
              actualizar: permiso.Actualizar,
              eliminar: permiso.Eliminar,
            }
          : { crear: false, leer: false, actualizar: false, eliminar: false };
        return acc;
      }, {});

      this.permisosIniciales = JSON.parse(JSON.stringify(this.permisos)); // Guardamos estado inicial
    },

    // ✅ Verificar si hubo cambios antes de actualizar
    confirmarGuardado() {
      const cambios = Object.keys(this.permisos).some(
        (moduloId) =>
          JSON.stringify(this.permisos[moduloId]) !== JSON.stringify(this.permisosIniciales[moduloId])
      );

      if (cambios) {
        if (confirm("⚠️ Se han modificado permisos. ¿Desea continuar?")) {
          this.guardarPermisos();
        }
      } else {
        this.mensajeError = "⚠️ No hay cambios en los permisos.";
      }
    },

    // 📝 Guardar actualización de permisos
    async guardarPermisos() {
  if (!this.usuarioSeleccionado) {
    this.mensajeError = "⚠️ Debes seleccionar un usuario.";
    return;
  }

  try {
    const permisosActualizados = Object.keys(this.permisos).map((idModulo) => ({
      idModulo: parseInt(idModulo, 10),
      idUsuarioRol: parseInt(this.usuarioSeleccionado, 10),
      Crear: !!this.permisos[idModulo].crear,
      Actualizar: !!this.permisos[idModulo].actualizar,
      Eliminar: !!this.permisos[idModulo].eliminar,
      Leer: !!this.permisos[idModulo].leer,
      Reportes: !!this.permisos[idModulo].reportes,
    }));

    console.log("📤 Enviando permisos actualizados:", permisosActualizados);

    await axios.put(
      `http://localhost:3001/api/permiso/${this.usuarioSeleccionado}`,
      { permisos: permisosActualizados },
      { withCredentials: true }
    );

    console.log("✅ Permisos actualizados correctamente en backend.");

    this.mensajeExito = "✅ Permisos actualizados correctamente.";
    this.mensajeError = "";
    this.permisosIniciales = JSON.parse(JSON.stringify(this.permisos));
  } catch (error) {
    console.error("❌ Error al actualizar permisos:", error);
    this.mensajeError = "⚠️ No se pudo actualizar los permisos. Intenta de nuevo.";
    this.mensajeExito = "";
  }
},





  },
  mounted() {
    this.cargarDatos();
  },
};
</script>

<style scoped>
@import "@/assets/adminStyles.css";

/* 📌 Estilos de la Tabla */
.permissions-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 20px;
}

.permissions-table th,
.permissions-table td {
  padding: 12px;
  text-align: center;
  border-bottom: 1px solid #ddd;
}

.permissions-table th {
  background: #002f5b;
  color: white;
}

/* 📌 Estilos para Select */
select {
  width: 100%;
  padding: 5px;
  border-radius: 5px;
  font-size: 1em;
}

/* 📌 Estilos de Mensajes */
.error-message {
  color: red;
  font-size: 0.9em;
  margin-top: 10px;
}

.success-message {
  color: green;
  font-size: 0.9em;
  margin-top: 10px;
}
</style>
